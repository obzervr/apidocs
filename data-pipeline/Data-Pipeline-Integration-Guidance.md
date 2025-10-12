# Obzervr Data Pipeline Integration Guidance

## Overview

This document provides technical guidance for integrating with the Obzervr Data Pipeline to synchronize data from Obzervr into external data stores and analytics systems for reporting purposes.

### Purpose
Data developers and integration developers can use this specification to understand how to extract data from Obzervr using the pipeline stored procedures, enabling downstream analytics and reporting solutions.

### Target Audience
- Data Engineers
- Integration Developers
- Analytics Developers
- Business Intelligence Developers

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Authentication & Rate Limiting](#authentication--rate-limiting)
3. [Common Parameters](#common-parameters)
4. [Data Extraction Patterns](#data-extraction-patterns)
5. [Entity Catalog](#entity-catalog)
6. [Error Handling](#error-handling)
7. [Best Practices](#best-practices)
8. [Code Examples](#code-examples)

---

## Architecture Overview

### Data Pipeline Design

The Obzervr Data Pipeline provides read-only access to operational data through a set of stored procedures in the `pipeline` schema. All procedures follow consistent patterns for:

- **Pagination**: Using `PagingId` (rowversion) or `Id` columns for efficient data retrieval
- **Incremental Loading**: Optional Date-based filtering via `LastUpdatedStart` and `LastUpdatedEnd`
- **Multi-tenancy**: Tenant filtering via `TenantIdList`
- **Rate Limiting**: Monitored work rates to prevent system overload

### Key Concepts

| Concept | Description |
|---------|-------------|
| **PagingId** | A SQL Server `rowversion` data type used for ordering and pagination. Guaranteed to be monotonically increasing. |
| **Soft Delete** | Some entities support soft deletes where `IsDeleted = 1` indicates logical deletion |
| **Insert Only** | Some fact tables are append-only (no updates), simplifying CDC logic |
| **Snapshot Tables** | Point-in-time snapshots of data, captured at specific intervals |

---

## Authentication & Rate Limiting

### Work rate Management

**All stored procedures enforce rate limiting** through resource usage and duration monitoring.

### Rate Limit Error

```sql
-- Error thrown when rate limited
THROW 50001, 'Request rate limited. Please follow rate limiting guidance to avoid rate limits.', 1;
```

**Response to Rate Limiting:**
- Implement exponential backoff
- Review your extraction frequency
- Contact Obzervr support for lease configuration adjustments

---

## Common Parameters

All stored procedures in the pipeline schema share a consistent parameter structure:

### Standard Parameters

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `@TenantIdList` | `nvarchar(max)` | `NULL` | No | Comma-separated list of Tenant IDs to filter. `NULL` returns all tenants. |
| `@RowCount` | `int` | Varies | No | Maximum rows to return per call. See entity-specific limits. |
| `@LastUpdatedStart` | `datetime` | `NULL` | No | Filter for records modified after this timestamp (exclusive). |
| `@LastUpdatedEnd` | `datetime` | `NULL` | No | Filter for records modified before this timestamp (inclusive). |
| `@LastPagingId` | `rowversion` | `NULL` | No | Resume pagination from this PagingId. Use for incremental loads. |
| `@LastId` | `uniqueidentifier` | `NULL` | No | Alternative paging column for some entities using GUID IDs. |

### Parameter Notes

**@TenantIdList:**
```sql
-- Single tenant
EXEC pipeline.sp_GetAssignments @TenantIdList = 'TENANT-001-GUID'

-- Multiple tenants
EXEC pipeline.sp_GetAssignments @TenantIdList = 'TENANT-001-GUID,TENANT-002-GUID,TENANT-003-GUID'

-- All tenants (default)
EXEC pipeline.sp_GetAssignments @TenantIdList = NULL
```

To retrieve a list of your tenants and their details:
```sql
EXEC pipeline.sp_GetTenants
```

**@RowCount:**
- Procedures enforce minimum and maximum bounds
- Exceeding max will automatically cap to the maximum
- See [Entity Catalog](#entity-catalog) for per-entity limits

**@LastPagingId:**
- Hexadecimal representation of rowversion: `0x0000000000003030`
- Use the `PagingId` from the last row of previous batch
- Essential for incremental and historical loads

---

## Data Extraction Patterns

### Pattern 1: Full Historical Load

Use for initial data warehouse population or full refreshes.

```sql
DECLARE @PagingId rowversion = NULL;
DECLARE @BatchSize int = 10000;

-- Loop until no more records
WHILE 1=1
BEGIN
    EXEC pipeline.sp_GetAssignments
        @TenantIdList = NULL,
        @RowCount = @BatchSize,
        @LastPagingId = @PagingId;
    
    -- Store results in temp table/staging area
    -- Get max PagingId from result set
    SET @PagingId = (SELECT MAX(PagingId) FROM #ResultSet);
    
    -- Exit if no more records
    IF @@ROWCOUNT = 0 OR @@ROWCOUNT < @BatchSize
        BREAK;
END
```

### Pattern 2: Incremental Load (Date-Based)

Use for regular ETL jobs to capture changes since last load.

```sql
DECLARE @LastLoadTime datetime = '2025-10-11 00:00:00';
DECLARE @CurrentLoadTime datetime = GETDATE();
DECLARE @PagingId rowversion = NULL;

-- Extract all changes in time window
EXEC pipeline.sp_GetAssignments
    @LastUpdatedStart = @LastLoadTime,
    @LastUpdatedEnd = @CurrentLoadTime,
    @LastPagingId = @PagingId,
    @RowCount = 100000;

-- Store @CurrentLoadTime for next run
```

### Pattern 3: Incremental Load (PagingId-Based)

Use when LastUpdated timestamp might not be reliable or for insert-only tables.

```sql
DECLARE @LastKnownPagingId rowversion = 0x0000000000003030;

EXEC pipeline.sp_GetAssignmentDetailsSnapshot
    @LastPagingId = @LastKnownPagingId,
    @RowCount = 100000;

-- Store max PagingId from results for next run
```

### Pattern 4: Tenant-Specific Extraction

Use for multi-tenant architectures where data is segregated by tenant.

```sql
-- Extract for specific tenant
EXEC pipeline.sp_GetAssignments
    @TenantIdList = 'TENANT-001',
    @LastUpdatedStart = '2025-10-01',
    @RowCount = 100000;
```

---

## Entity Catalog

The following tables are available through the data pipeline, organized by size classification.

### Extra Large Entities

High-volume transactional data requiring careful pagination.

| Table Name | Stored Procedure | Default Page Size | Max Page Size | Soft Delete | Insert Only | Notes |
|------------|------------------|-------------------|---------------|-------------|-------------|-------|
| **FactAssignmentDetailsSnapshot** | `pipeline.sp_GeAssignmentDetailsSnapshot` | 1,000 | 100,000 | No | Yes | Assignment details at point in time |
| **FactAssignmentProgressSnapshot** | `pipeline.sp_GetAssignmentProgressSnapshot` | 1,000 | 100,000 | No | Yes | Work order completion percentages. Limited analytics use case. |
| **FactAuditsUserAssignment** | `pipeline.sp_GetAuditsUserAssignment` | 1,000 | 100,000 | No | Yes | User assignment audit trail |
| **FactAuditCommandCountSnapshotHourly** | `pipeline.sp_GetAuditCommandCountSnapshotHourly`<br/>`pipeline.sp_GetFactAuditCommandCountSnapshotHourly` | 1,000 | 100,000 | No | Yes | Hourly command usage statistics |
| **FactTimeSeriesFieldMeasurements** | `pipeline.sp_GetTimeSeriesFieldMeasurements` | 1,000 | 1,000,000 | Yes | No | Time-series field measurements with soft delete support |
| **FactTimeSeries** | `pipeline.sp_GetTimeSeries` | 1,000 | 100,000 | Yes | No | Time-series parent records |

#### Example: FactAssignmentDetailsSnapshot

```sql
-- Incremental load with pagination
EXEC pipeline.sp_GeAssignmentDetailsSnapshot
    @LastPagingId = 0x0000000000003030,
    @LastUpdatedStart = '2025-10-01',
    @RowCount = 100000;
```

**Key Columns:**
- `TenantId`: Tenant identifier
- `AssignmentId`: Reference to parent assignment
- `UserId`, `TeamId`: Assignment allocation
- `Status`: Current assignment status
- `SnapshotTimestamp`: When snapshot was captured
- `PagingId`: Rowversion for pagination
- `LastLoaded`: ETL timestamp for incremental loads

---

### Large Entities

Moderate-volume operational data.

| Table Name | Stored Procedure | Default Page Size | Max Page Size | Soft Delete | Insert Only | Notes |
|------------|------------------|-------------------|---------------|-------------|-------------|-------|
| **DimAssignments** | `pipeline.sp_GetAssignments` | 1,000 | 100,000 | Yes | No | Core work order/assignment master data |
| **FactAssignmentDeclinedReasons** | `pipeline.sp_GetAssignmentDeclinedReasons` | 1,000 | 100,000 | No | No | Reasons for declined assignments |
| **FactAssignmentExceptions** | `pipeline.sp_GetAssignmentExceptions` | 1,000 | 100,000 | No | No | Assignment exception records |
| **FactAssignmentTags** | `pipeline.sp_GetAssignmentTags` | 1,000 | 100,000 | Yes | No | Tags/labels associated with assignments |
| **FactAssignmentFieldOperators** | `pipeline.sp_GetAssignmentFieldOperators` | 1,000 | 100,000 | Yes | No | Field operator assignments |

#### Example: DimAssignments

```sql
-- Full dimension load with soft delete handling
EXEC pipeline.sp_GetAssignments
    @TenantIdList = 'TENANT-001',
    @LastUpdatedStart = '2025-10-01',
    @RowCount = 50000;
```

**Key Columns:**
- `Id`: Unique assignment identifier (GUID)
- `TenantId`: Tenant identifier
- `AssignmentCode`: Business key
- `Status`: Assignment workflow status
- `AssignedTo`: User assigned
- `CreatedDate`, `LastUpdated`: Audit timestamps
- `IsDeleted`: Soft delete flag
- `PagingId`: For pagination

---

### Medium Entities

Entities with moderate data volumes, may vary by customer.

| Table Name | Stored Procedure | Default Page Size | Max Page Size | Notes |
|------------|------------------|-------------------|---------------|-------|
| **DimAssignmentPoints** | `pipeline.sp_GetAssignmentPoints` | 1,000 | 100,000 | Large/Extra Large for some customers |
| **FactAssignmentPointAttributes** | `pipeline.sp_GetAssignmentPointAttributes` | 1,000 | 100,000 | Large/Extra Large for some customers |
| **FactTimeSeriesFieldMeasurementDocuments** | `pipeline.sp_GetTimeSeriesFieldMeasurementDocuments` | 1,000 | 100,000 | Document attachments (photographs, videos, files, etc.) for measurements |
| **FactTimeSeriesSectionDocuments** | `pipeline.sp_GetTimeSeriesSectionDocuments` | 1,000 | 10,000 | Section-level documents |

---

### Small Entities

Reference data, configuration tables, and lookup tables.

| Table Name | Stored Procedure | Default Page Size | Max Page Size | Paging Required | Notes |
|------------|------------------|-------------------|---------------|-----------------|-------|
| **DimAssignmentCategories** | `pipeline.sp_GetAssignmentCategories` | 100 | 100 | No | Assignment categorization |
| **DimAssignmentStatus** | `pipeline.sp_GetAssignmentStatus` | 10 | 10 | No | Status lookup table |
| **DimSites** | `pipeline.sp_GetSites` | 100 | 100 | No | Site/location master |
| **DimSubSites** | `pipeline.sp_GetSubSites` | 100 | 100 | No | Sub-location master |
| **DimTeams** | `pipeline.sp_GetTeams` | 100 | 100 | Yes | Team definitions |
| **DimWorkTemplates** | `pipeline.sp_GetWorkTemplates` | 1,000 | 1,000 | Yes | Work template definitions |
| **DimTemplateGroups** | `pipeline.sp_GetTemplateGroups` | 100 | 100 | No | Template groupings |
| **FactTemplateGroupWorkTemplates** | `pipeline.sp_GetTemplateGroupWorkTemplates` | 100 | 100 | Yes | Template group associations |
| **LabelAlias** | `pipeline.sp_GetLabelAlias` | 100 | 100 | No | Label/terminology configuration |
| **TenantSettings** | `pipeline.sp_GetTenants` | 10 | 10 | No | Tenant configuration |
| **DimUsers** | `pipeline.sp_GetUsers` | 1,000 | 1,000 | Yes | User master data |
| **FactTeamUsers** | `pipeline.sp_GetTeamUsers` | 1,000 | 1,000 | Yes | Team membership |
| **FactFieldMeasurementTables** | `pipeline.sp_GetFieldMeasurementTableContents` | 10,000 | 10,000 | Yes | Table measurement data |
| **DimFieldMeasurementTables** | `pipeline.sp_GetFieldMeasurementTableDefinitions` | 100 | 100 | Yes | Table definitions |

#### Example: Small Entity (Lookup Table)

```sql
-- Simple full load - no pagination needed
EXEC pipeline.sp_GetAssignmentStatus;

-- Result: ~10 rows or less
```

---

## Error Handling

### Rate Limit Error (50001)

**Cause:** No valid lease or rate limit exceeded

**Error Message:**
```
Request rate limited. Please follow rate limiting guidance to avoid rate limits.
```

**Resolution:**
1. Implement exponential backoff (start with 30 seconds, double each retry)
2. Check with Obzervr administrator for lease configuration
3. Reduce extraction frequency
4. Optimize batch sizes

**Example Error Handling:**

```sql
BEGIN TRY
    EXEC pipeline.sp_GetAssignments
        @RowCount = 50000,
        @LastPagingId = @LastPagingId;
END TRY
BEGIN CATCH
    IF ERROR_NUMBER() = 50001
    BEGIN
        -- Log rate limit error
        PRINT 'Rate limited. Waiting before retry...';
        WAITFOR DELAY '00:00:30'; -- 30 second delay
        -- Implement retry logic here
    END
    ELSE
    BEGIN
        -- Handle other errors
        THROW;
    END
END CATCH
```

### General Error Handling Pattern

```sql
BEGIN TRY
    -- Your extraction logic
    EXEC pipeline.sp_GetAssignments @RowCount = 50000;
    
    -- Process results
    
END TRY
BEGIN CATCH
    -- Log error details
    DECLARE @ErrorMessage nvarchar(4000) = ERROR_MESSAGE();
    DECLARE @ErrorNumber int = ERROR_NUMBER();
    DECLARE @ErrorSeverity int = ERROR_SEVERITY();
    
    -- Log to your ETL framework
    INSERT INTO ETL.ErrorLog (ErrorNumber, ErrorMessage, ErrorSeverity, ProcedureName, ErrorTime)
    VALUES (@ErrorNumber, @ErrorMessage, @ErrorSeverity, 'pipeline.sp_GetAssignments', GETDATE());
    
    -- Re-throw if critical
    IF @ErrorSeverity > 10
        THROW;
END CATCH
```

---

## Best Practices

This section provides critical guidance for reliable data extraction from the Obzervr Data Pipeline.

**Key Topics:**
1. **Pagination Strategy** - How to page through large result sets
2. **Incremental Loading** - Two strategies based on entity support (PagingId vs LastId)
3. **Upsert/Merge Pattern** - Critical for tables that support updates (non-insert-only)
4. **Batch Sizing** - Optimal page sizes by entity classification
5. **Soft Delete Handling** - Managing logical deletes
6. **Multi-Tenant Considerations** - Extracting single vs. multiple tenants

### 1. Pagination Strategy

- Always use `PagingId` for pagination, not offset-based pagination
- Store the last `PagingId` after each batch
- Results are always ordered by `PagingId` for consistent pagination

### 2. Incremental Loading Strategies

Choose the appropriate strategy based on the paging parameter available for the entity.

> **Quick Decision:** Check the [Entity Catalog](#entity-catalog) to see which parameter your entity supports:
> - Most entities support `@LastPagingId` → Use **Strategy A** (simpler, recommended)
> - Some entities only support `@LastId` → Use **Strategy B** (requires more care)

#### Strategy A: PagingId-Based Incremental Loading (Preferred)

**When to use:** Entity supports `@LastPagingId` parameter (most entities)

**How it works:**
- Use `@LastPagingId` for both incremental loading AND pagination within a run
- No need for `@LastUpdatedStart` or `@LastUpdatedEnd` unless you have specific business requirements to filter by time period
- Simply pass the last `PagingId` from your previous run and continue from there

**Example:**
```sql
-- Day 1: Initial load
EXEC pipeline.sp_GetAssignments
    @RowCount = 50000,
    @LastPagingId = NULL;
-- Store max PagingId: 0x0000000000003030

-- Day 2 or for next page in Day 1: Continue from where you left off
EXEC pipeline.sp_GetAssignments
    @RowCount = 50000,
    @LastPagingId = 0x0000000000003030;
-- Get next batch automatically
```

**Advantages:**
- Simplest approach
- No concern about records modified during extraction
- Efficient rowversion-based pagination
- Works for both incremental and pagination needs

#### Strategy B: LastId + Timestamp Incremental Loading

**When to use:** Entity uses `@LastId` (GUID) parameter instead of `@LastPagingId`

**How it works:**
- Use `@LastUpdatedStart` to define the incremental window (e.g., daily runs)
- Use `@LastId` for pagination WITHIN each run
- **IMPORTANT:** Keep `@LastUpdatedStart` constant across all pages within a single run
- Results are ordered by `LastId`, NOT by `LastUpdated`

**Example - Daily Synchronization:**

```sql
-- ============================================
-- DAY 1: July 11, 2025
-- ============================================

-- Get ETL metadata: last successful run was July 10, 2025 at 23:59:59
DECLARE @LastRunTime datetime = '2025-07-10 23:59:59';
DECLARE @CurrentRunTime datetime = '2025-07-11 23:59:59';
DECLARE @LastId uniqueidentifier = NULL;
DECLARE @RowCount int = 50000;

-- Page 1: First page of today's incremental
EXEC pipeline.sp_GetSomeEntity
    @LastUpdatedStart = @LastRunTime,        -- Filter: records since last run
    @LastUpdatedEnd = @CurrentRunTime,       -- Filter: up to current run
    @LastId = NULL,                          -- Start from beginning of this time window
    @RowCount = @RowCount;

-- Result: 50,000 rows returned (hit page size limit)
-- Last row has LastId = 'A1B2C3D4-...' and LastLoaded = '2025-07-11 14:30:22'

-- Page 2: Get next page for SAME incremental window
EXEC pipeline.sp_GetSomeEntity
    @LastUpdatedStart = @LastRunTime,        -- SAME VALUE - don't change!
    @LastUpdatedEnd = @CurrentRunTime,       -- SAME VALUE - don't change!
    @LastId = 'A1B2C3D4-...',               -- Resume pagination using LastId
    @RowCount = @RowCount;

-- Result: 50,000 more rows
-- Last row has LastId = 'E5F6G7H8-...'

-- Page 3: Continue until all pages exhausted
EXEC pipeline.sp_GetSomeEntity
    @LastUpdatedStart = @LastRunTime,        -- STILL SAME VALUE
    @LastUpdatedEnd = @CurrentRunTime,       -- STILL SAME VALUE
    @LastId = 'E5F6G7H8-...',               -- Next page
    @RowCount = @RowCount;

-- Result: 15,000 rows (less than page size = done)
-- All changes from Jul 10 to Jul 11 now extracted

-- ============================================
-- DAY 2: July 12, 2025
-- ============================================

-- Start NEW incremental run for next day
SET @LastRunTime = '2025-07-11 23:59:59';   -- Updated to previous run
SET @CurrentRunTime = '2025-07-12 23:59:59';
SET @LastId = NULL;                          -- Reset for new run

-- Page 1 of new day
EXEC pipeline.sp_GetSomeEntity
    @LastUpdatedStart = @LastRunTime,        -- New time window
    @LastUpdatedEnd = @CurrentRunTime,
    @LastId = NULL,                          -- Start fresh for this window
    @RowCount = @RowCount;

-- Continue paging as needed for Day 2 data...
```

**Critical Rules for Strategy B:**

1. **Between Runs (Day to Day):**
   - Update `@LastUpdatedStart` to the timestamp from the previous successful run
   - Reset `@LastId` to NULL

2. **Within a Run (Paging):**
   - Keep `@LastUpdatedStart` and `@LastUpdatedEnd` constant
   - Only change `@LastId` to paginate through results
   - Never use `LastUpdated/LastLoaded` values for pagination

3. **Result Ordering:**
   - Results are ordered by `LastId` (GUID), not by `LastUpdated` timestamp
   - This is why you must use `LastId` for pagination, not timestamp values

**Why this matters:**
```sql
-- ❌ WRONG: Trying to use LastUpdated for pagination
EXEC pipeline.sp_GetSomeEntity
    @LastUpdatedStart = '2025-07-11 14:30:22',  -- Using last row's timestamp
    @RowCount = 50000;
-- Problem: You'll miss records or get duplicates because results aren't ordered by LastUpdated!

-- ✅ CORRECT: Using LastId for pagination
EXEC pipeline.sp_GetSomeEntity
    @LastUpdatedStart = '2025-07-11 00:00:00',  -- Original window start (unchanged)
    @LastId = 'A1B2C3D4-...',                   -- Last row's ID
    @RowCount = 50000;
-- Results continue correctly in LastId order
```

**Storage Requirements:**

For Strategy B, store in your ETL metadata table:
- `LastSuccessfulRunTime`: The `@LastUpdatedEnd` value from completed run
- Do NOT store `LastId` between runs (reset to NULL each run)
- Store `LastId` only temporarily during pagination within a single run

#### Choosing the Right Strategy

**Use Strategy A (PagingId) when:**
- ✅ The entity supports `@LastPagingId` parameter
- ✅ You want the simplest, most reliable approach
- ✅ You need guaranteed consistency across all tenants
- ✅ You want to avoid timestamp-related edge cases

**Use Strategy B (LastId + Timestamp) when:**
- ⚠️ The entity only supports `@LastId` parameter (no PagingId available)
- ⚠️ You have specific business requirements for time-window filtering
- ⚠️ You're willing to implement robust deduplication in your target

**Recommendation:** Use Strategy A (PagingId) whenever possible. Strategy B is provided for entities that don't support PagingId.

#### Strategy Comparison

| Aspect | Strategy A (PagingId) | Strategy B (LastId + Timestamp) |
|--------|----------------------|----------------------------------|
| **Complexity** | ✅ Simple | ⚠️ Complex |
| **Parameters Used** | `@LastPagingId`, `@RowCount` | `@LastUpdatedStart`, `@LastUpdatedEnd`, `@LastId`, `@RowCount` |
| **Incremental & Paging** | ✅ Single parameter handles both | ⚠️ Timestamp for incremental, LastId for paging |
| **Multi-Tenant Safety** | ✅ Fully consistent | ⚠️ Requires scheduling consideration |
| **Duplicate Risk** | ✅ Minimal | ⚠️ Yes - requires MERGE/UPSERT |
| **Load Timing Sensitivity** | ✅ No | ⚠️ Yes - extract after loads complete |
| **Best For** | Most use cases | Entities without PagingId support |

#### Strategy B: Important Considerations

When using Strategy B (LastId + Timestamp), be aware of these important considerations:

##### ⚠️ Duplicate Handling Required

Records may appear in multiple extractions due to updates in the Analytics store:

```sql
-- Record loaded at 10:00:00
-- Customer extraction at 10:02:00 captures it

-- Record updated at 10:05:00 (LastLoaded changes)
-- Customer's next extraction at 10:10:00 captures it AGAIN

-- Result: Duplicate in customer's data warehouse
```

**Solution:** Implement proper upsert/merge logic in your data warehouse.

> **See [Upsert/Merge Pattern](#3-upsertmerge-pattern-critical-for-non-insert-only-tables)** for detailed implementation examples in SQL Server, PostgreSQL, Snowflake, and Python.

##### ⚠️ Multi-Tenant Extraction Timing

When extracting all tenants (`@TenantIdList = NULL`), tenant data loads sequentially into the Analytics store:

```sql
-- Analytics Store Load Schedule (example):
-- 08:00-08:30 - Tenant A loads
-- 08:30-09:00 - Tenant B loads  
-- 09:00-09:30 - Tenant C loads

-- If you extract at 08:45:
-- ✅ Get complete Tenant A data
-- ❌ Miss Tenant B data (not loaded yet)
-- ❌ Miss Tenant C data (not loaded yet)
```

**Solutions:**

1. **Schedule extractions after all tenant loads complete** (Recommended)
```sql
-- Consult with Obzervr administrator for load completion times
-- If loads complete by 10:00, schedule extractions at 11:00

EXEC pipeline.sp_GetAssignments
    @LastUpdatedStart = '2025-07-11 11:00:00',
    @LastUpdatedEnd = '2025-07-12 11:00:00',
    @RowCount = 50000;
```

2. **Extract tenants separately**
```sql
-- Separate job for each tenant, scheduled after that tenant's load
-- Job 1: After Tenant A completes (08:30)
EXEC pipeline.sp_GetAssignments
    @TenantIdList = 'TENANT-A-GUID',
    @LastUpdatedStart = @LastRun,
    @RowCount = 50000;

-- Job 2: After Tenant B completes (09:00)
EXEC pipeline.sp_GetAssignments
    @TenantIdList = 'TENANT-B-GUID',
    @LastUpdatedStart = @LastRun,
    @RowCount = 50000;
```

##### ⚠️ Extraction During Load Windows

Best practice: Schedule extractions during periods when Analytics store loads are NOT running.

```sql
-- ✅ GOOD: Extract during quiet period
-- Loads: 08:00-10:00
-- Extract: 11:00 (after loads complete)

DECLARE @LastRunTime datetime = '2025-07-11 11:00:00';
DECLARE @CurrentRunTime datetime = '2025-07-12 11:00:00';

EXEC pipeline.sp_GetSomeEntity
    @LastUpdatedStart = @LastRunTime,
    @LastUpdatedEnd = @CurrentRunTime,
    @LastId = NULL,
    @RowCount = 50000;

-- This eliminates risks of:
-- - Partial tenant data
-- - Mid-extraction data arrivals
-- - Inconsistent point-in-time views
```

**Contact your Obzervr administrator** to understand:
- Analytics store load schedules
- Recommended extraction windows
- Per-tenant load completion times

### 3. Upsert/Merge Pattern (Critical for Non-Insert-Only Tables)

**When to use:** All entities that are NOT marked as "Insert Only" in the [Entity Catalog](#entity-catalog)

Records may appear in multiple extractions when:
- Using **Strategy A (PagingId)**: Record is updated in Analytics store, gets new PagingId
- Using **Strategy B (LastId + Timestamp)**: Record is updated in Analytics store, gets new LastLoaded timestamp
- The entity supports updates (not Insert Only)

**Why this matters:**

```sql
-- Initial extraction captures Assignment ABC
-- Assignment ABC: Status = 'In Progress', PagingId = 0x1000

-- Later, assignment is completed
-- Assignment ABC: Status = 'Complete', PagingId = 0x2000 (new!)

-- Next extraction with @LastPagingId = 0x1000
-- Gets Assignment ABC again with Status = 'Complete'

-- Without proper handling: TWO records in your warehouse ❌
-- With upsert/merge: ONE record, updated status ✅
```

#### Implementation Examples

**SQL Server (MERGE):**
```sql
-- Stage extracted data first
CREATE TABLE Staging.DimAssignments (
    Id uniqueidentifier PRIMARY KEY,
    TenantId nvarchar(50),
    AssignmentCode nvarchar(100),
    Status nvarchar(50),
    LastUpdated datetime2,
    IsDeleted bit,
    PagingId rowversion
);

-- Load from stored procedure into staging
INSERT INTO Staging.DimAssignments
EXEC pipeline.sp_GetAssignments 
    @LastPagingId = @LastPagingId,
    @RowCount = 50000;

-- Merge staging into target
MERGE INTO DataWarehouse.DimAssignments AS target
USING Staging.DimAssignments AS source
ON target.Id = source.Id
WHEN MATCHED THEN 
    UPDATE SET 
        target.TenantId = source.TenantId,
        target.AssignmentCode = source.AssignmentCode,
        target.Status = source.Status,
        target.LastUpdated = source.LastUpdated,
        target.IsDeleted = source.IsDeleted,
        target.PagingId = source.PagingId,
        target.ETL_UpdatedAt = GETUTCDATE()
WHEN NOT MATCHED THEN
    INSERT (Id, TenantId, AssignmentCode, Status, LastUpdated, IsDeleted, PagingId, ETL_CreatedAt, ETL_UpdatedAt)
    VALUES (source.Id, source.TenantId, source.AssignmentCode, source.Status, 
            source.LastUpdated, source.IsDeleted, source.PagingId, GETUTCDATE(), GETUTCDATE());

-- Clean up staging
TRUNCATE TABLE Staging.DimAssignments;
```

**PostgreSQL (INSERT ... ON CONFLICT):**
```sql
-- Load into staging table
CREATE TEMP TABLE staging_assignments AS
SELECT * FROM dblink('obzervr_connection',
    'EXEC pipeline.sp_GetAssignments @LastPagingId = 0x1000, @RowCount = 50000')
AS t(...);

-- Upsert into target
INSERT INTO data_warehouse.dim_assignments (
    id, tenant_id, assignment_code, status, last_updated, is_deleted, paging_id, etl_created_at, etl_updated_at
)
SELECT 
    id, tenant_id, assignment_code, status, last_updated, is_deleted, paging_id,
    NOW(), NOW()
FROM staging_assignments
ON CONFLICT (id) 
DO UPDATE SET 
    tenant_id = EXCLUDED.tenant_id,
    assignment_code = EXCLUDED.assignment_code,
    status = EXCLUDED.status,
    last_updated = EXCLUDED.last_updated,
    is_deleted = EXCLUDED.is_deleted,
    paging_id = EXCLUDED.paging_id,
    etl_updated_at = NOW();
```

**Snowflake (MERGE):**
```sql
-- Create staging table
CREATE OR REPLACE TEMP TABLE staging_assignments LIKE data_warehouse.dim_assignments;

-- Load from external stage or directly
INSERT INTO staging_assignments
SELECT * FROM TABLE(result_scan(last_query_id()));

-- Merge
MERGE INTO data_warehouse.dim_assignments AS target
USING staging_assignments AS source
ON target.id = source.id
WHEN MATCHED THEN 
    UPDATE SET 
        target.tenant_id = source.tenant_id,
        target.assignment_code = source.assignment_code,
        target.status = source.status,
        target.last_updated = source.last_updated,
        target.is_deleted = source.is_deleted,
        target.paging_id = source.paging_id,
        target.etl_updated_at = CURRENT_TIMESTAMP()
WHEN NOT MATCHED THEN
    INSERT (id, tenant_id, assignment_code, status, last_updated, is_deleted, paging_id, etl_created_at, etl_updated_at)
    VALUES (source.id, source.tenant_id, source.assignment_code, source.status, 
            source.last_updated, source.is_deleted, source.paging_id, CURRENT_TIMESTAMP(), CURRENT_TIMESTAMP());
```

**Python with pandas (DataFrame upsert):**
```python
import pandas as pd
from sqlalchemy import create_engine

# Extract data
df = pd.read_sql(
    "EXEC pipeline.sp_GetAssignments @LastPagingId = ?, @RowCount = ?",
    engine,
    params=[last_paging_id, 50000]
)

# Upsert pattern depends on your target database
# For PostgreSQL with SQLAlchemy:
df.to_sql(
    'dim_assignments',
    engine,
    schema='data_warehouse',
    if_exists='append',  # Use 'append' with on_conflict
    index=False,
    method='multi'
)

# Or use explicit UPDATE/INSERT logic:
for _, row in df.iterrows():
    engine.execute("""
        INSERT INTO data_warehouse.dim_assignments (...)
        VALUES (...)
        ON CONFLICT (id) DO UPDATE SET ...
    """, row.to_dict())
```

#### Critical Rules for Upsert/Merge

1. **Always use the entity's primary key** (usually `Id` column) for matching
2. **Update ALL columns** on match, don't skip columns
3. **Include ETL audit columns** (`ETL_CreatedAt`, `ETL_UpdatedAt`) to track changes
4. **Test your merge logic** with known duplicates before production
5. **Monitor for merge failures** - they indicate data quality issues

#### Insert-Only Tables

For tables marked as **"Insert Only" = Yes** in the Entity Catalog:
```sql
-- Simple INSERT is sufficient (no merge needed)
INSERT INTO DataWarehouse.FactAssignmentDetailsSnapshot
EXEC pipeline.sp_GeAssignmentDetailsSnapshot
    @LastPagingId = @LastPagingId,
    @RowCount = 100000;

-- These tables are append-only, no updates occur
```

Refer to the [Entity Catalog](#entity-catalog) to identify which tables are Insert Only.

### 4. Batch Sizing

**Recommended Batch Sizes by Entity Classification:**

| Classification | Recommended Batch Size | Notes |
|----------------|------------------------|-------|
| Extra Large | 10,000 - 50,000 | Balance between throughput and memory |
| Large | 50,000 - 100,000 | Can use max for most cases |
| Medium | 10,000 - 100,000 | Depends on customer data volume |
| Small | Maximum allowed | Usually completes in single call |

### 5. Soft Delete Handling

For entities with `IsDeleted` flag:

```sql
-- Include deleted records in your extraction
EXEC pipeline.sp_GetAssignments
    @LastUpdatedStart = @LastLoadTime,
    @RowCount = 50000;

-- In your target system, implement Type 2 SCD or soft delete pattern
-- DO NOT filter out IsDeleted = 1 in your extraction
-- Process deletes in your transformation layer
```

### 6. Multi-Tenant Considerations

**Option A: Single Job, All Tenants**
```sql
-- Extract all tenants together
EXEC pipeline.sp_GetAssignments 
    @TenantIdList = NULL,
    @RowCount = 100000;

-- Partition data by TenantId in your data warehouse
```

⚠️ **Important:** When using Strategy B (LastId + Timestamp) with all tenants:
- Schedule extractions after all tenant loads complete
- Alternatively, use Strategy A (PagingId) which handles multi-tenant timing automatically
- See [Strategy B: Important Considerations](#strategy-b-important-considerations) for details

**Option B: Separate Jobs per Tenant**
```sql
-- Create separate ETL jobs for each tenant
EXEC pipeline.sp_GetAssignments 
    @TenantIdList = 'TENANT-001',
    @RowCount = 50000;
```

✅ **Advantage:** Eliminates multi-tenant timing concerns with Strategy B
- Each job can be scheduled independently
- Align each job with that tenant's load completion time

## Code Examples

### Example 1: C# Integration (ADO.NET)

```csharp
using System;
using System.Data;
using System.Data.SqlClient;

public class ObzervDataPipeline
{
    private readonly string _connectionString;
    
    public ObzervDataPipeline(string connectionString)
    {
        _connectionString = connectionString;
    }
    
    public async Task ExtractAssignmentsIncremental(
        DateTime lastLoadTime, 
        byte[] lastPagingId = null)
    {
        using (var connection = new SqlConnection(_connectionString))
        {
            await connection.OpenAsync();
            
            using (var command = new SqlCommand("pipeline.sp_GetAssignments", connection))
            {
                command.CommandType = CommandType.StoredProcedure;
                command.CommandTimeout = 300; // 5 minutes
                
                // Add parameters
                command.Parameters.AddWithValue("@TenantIdList", DBNull.Value);
                command.Parameters.AddWithValue("@RowCount", 50000);
                command.Parameters.AddWithValue("@LastUpdatedStart", lastLoadTime);
                command.Parameters.AddWithValue("@LastUpdatedEnd", DateTime.UtcNow);
                
                if (lastPagingId != null)
                    command.Parameters.AddWithValue("@LastPagingId", lastPagingId);
                else
                    command.Parameters.AddWithValue("@LastPagingId", DBNull.Value);
                
                try
                {
                    using (var reader = await command.ExecuteReaderAsync())
                    {
                        while (await reader.ReadAsync())
                        {
                            // Process each row
                            var assignment = new Assignment
                            {
                                Id = reader.GetGuid(reader.GetOrdinal("Id")),
                                TenantId = reader.GetString(reader.GetOrdinal("TenantId")),
                                AssignmentCode = reader.GetString(reader.GetOrdinal("AssignmentCode")),
                                Status = reader.GetString(reader.GetOrdinal("Status")),
                                LastUpdated = reader.GetDateTime(reader.GetOrdinal("LastUpdated")),
                                IsDeleted = reader.GetBoolean(reader.GetOrdinal("IsDeleted")),
                                PagingId = (byte[])reader["PagingId"]
                            };
                            
                            // Insert/Update in your data warehouse
                            await UpsertAssignment(assignment);
                        }
                    }
                }
                catch (SqlException ex) when (ex.Number == 50001)
                {
                    // Handle rate limiting
                    Console.WriteLine("Rate limited. Implementing backoff...");
                    await Task.Delay(TimeSpan.FromSeconds(30));
                    throw; // Retry in calling code
                }
            }
        }
    }
    
    private async Task UpsertAssignment(Assignment assignment)
    {
        // Your data warehouse upsert logic
    }
}

public class Assignment
{
    public Guid Id { get; set; }
    public string TenantId { get; set; }
    public string AssignmentCode { get; set; }
    public string Status { get; set; }
    public DateTime LastUpdated { get; set; }
    public bool IsDeleted { get; set; }
    public byte[] PagingId { get; set; }
}
```

### Example 2: Python Integration (pyodbc)

```python
import pyodbc
import time
from datetime import datetime, timedelta

class ObzervPipeline:
    def __init__(self, connection_string):
        self.connection_string = connection_string
    
    def extract_assignments_incremental(self, last_load_time, last_paging_id=None):
        """
        Extract assignments using incremental pattern
        """
        retry_count = 0
        max_retries = 3
        
        while retry_count < max_retries:
            try:
                with pyodbc.connect(self.connection_string, timeout=300) as conn:
                    cursor = conn.cursor()
                    
                    # Call stored procedure
                    cursor.execute("""
                        EXEC pipeline.sp_GetAssignments
                            @TenantIdList = ?,
                            @RowCount = ?,
                            @LastUpdatedStart = ?,
                            @LastUpdatedEnd = ?,
                            @LastPagingId = ?
                    """, 
                    None,  # TenantIdList - NULL for all
                    50000,  # RowCount
                    last_load_time,
                    datetime.utcnow(),
                    last_paging_id)
                    
                    rows = cursor.fetchall()
                    columns = [column[0] for column in cursor.description]
                    
                    # Process results
                    for row in rows:
                        record = dict(zip(columns, row))
                        self.upsert_assignment(record)
                    
                    # Get max PagingId for next batch
                    if rows:
                        max_paging_id = rows[-1].PagingId
                        return max_paging_id
                    
                    return None
                    
            except pyodbc.Error as ex:
                sqlstate = ex.args[0] if ex.args else None
                
                # Check for rate limit error (50001)
                if sqlstate and '50001' in str(ex):
                    retry_count += 1
                    wait_time = 30 * (2 ** retry_count)  # Exponential backoff
                    print(f"Rate limited. Waiting {wait_time} seconds...")
                    time.sleep(wait_time)
                else:
                    raise
        
        raise Exception("Max retries exceeded due to rate limiting")
    
    def upsert_assignment(self, record):
        """Insert or update assignment in data warehouse"""
        # Your data warehouse logic here
        pass

# Usage
pipeline = ObzervPipeline("Driver={ODBC Driver 17 for SQL Server};Server=...")
last_load = datetime.utcnow() - timedelta(days=1)
pipeline.extract_assignments_incremental(last_load)
```

### Example 3: SSIS Pattern (T-SQL)

```sql
-- Step 1: Control Flow Variable Setup
DECLARE @LastLoadTime datetime;
DECLARE @CurrentLoadTime datetime;
DECLARE @LastPagingId rowversion;
DECLARE @MaxPagingId rowversion;
DECLARE @RowsProcessed int;
DECLARE @TotalRows int = 0;
DECLARE @BatchSize int = 50000;

-- Get last successful load time from control table
SELECT @LastLoadTime = LastSuccessfulLoad 
FROM ETL.ControlTable 
WHERE EntityName = 'DimAssignments';

SET @CurrentLoadTime = GETUTCDATE();
SET @LastPagingId = NULL;

-- Step 2: Extraction Loop
WHILE 1=1
BEGIN
    BEGIN TRY
        -- Truncate staging table
        TRUNCATE TABLE Staging.DimAssignments;
        
        -- Extract batch
        INSERT INTO Staging.DimAssignments
        EXEC pipeline.sp_GetAssignments
            @TenantIdList = NULL,
            @RowCount = @BatchSize,
            @LastUpdatedStart = @LastLoadTime,
            @LastUpdatedEnd = @CurrentLoadTime,
            @LastPagingId = @LastPagingId;
        
        SET @RowsProcessed = @@ROWCOUNT;
        SET @TotalRows = @TotalRows + @RowsProcessed;
        
        -- Process staging to target
        EXEC ETL.MergeDimAssignments;
        
        -- Get max PagingId for next iteration
        SELECT @MaxPagingId = MAX(PagingId) 
        FROM Staging.DimAssignments;
        
        SET @LastPagingId = @MaxPagingId;
        
        -- Exit if no more records
        IF @RowsProcessed = 0 OR @RowsProcessed < @BatchSize
            BREAK;
            
    END TRY
    BEGIN CATCH
        IF ERROR_NUMBER() = 50001
        BEGIN
            -- Rate limited - wait and retry
            WAITFOR DELAY '00:00:30';
            CONTINUE;
        END
        ELSE
        BEGIN
            -- Log error and exit
            INSERT INTO ETL.ErrorLog (ErrorMessage, ErrorTime)
            VALUES (ERROR_MESSAGE(), GETUTCDATE());
            THROW;
        END
    END CATCH
END

-- Step 3: Update control table on success
UPDATE ETL.ControlTable
SET LastSuccessfulLoad = @CurrentLoadTime,
    LastRowCount = @TotalRows,
    LastExecutionTime = GETUTCDATE()
WHERE EntityName = 'DimAssignments';

PRINT 'Extraction complete. Total rows: ' + CAST(@TotalRows AS nvarchar(20));
```

---

## ETL Architecture Recommendations

### Recommended Architecture Layers

```
┌─────────────────────────────────────────────────────────────┐
│                     Obzervr Database                        │
│                  (Source of Truth)                          │
│                pipeline.sp_Get* procedures                  │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      │ Rate Limited Extraction
                      ↓
┌─────────────────────────────────────────────────────────────┐
│              Staging Layer (Transient)                      │
│  - Exact copy of source structure                           │
│  - Truncate/Load pattern                                    │
│  - Minimal transformations                                  │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      │ Data Quality & Transformation
                      ↓
┌─────────────────────────────────────────────────────────────┐
│         Data Warehouse / Analytics Store                    │
│  - Dimensional model (Star/Snowflake)                       │
│  - Type 2 SCDs for historization                            │
│  - Conformed dimensions                                     │
│  - Aggregate tables                                         │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│           Analytics Layer (Power BI, Tableau, etc.)         │
└─────────────────────────────────────────────────────────────┘
```

### Scheduling Recommendations

| Entity Classification | Recommended Frequency | Notes |
|-----------------------|----------------------|-------|
| Extra Large (Facts) | Every 1-4 hours | Consider off-peak hours |
| Large (Facts & Dims) | Every 4-12 hours | Balance freshness vs load |
| Medium | Daily | Usually sufficient |
| Small (Reference) | Daily or Weekly | Changes infrequent |

---

## Troubleshooting Guide

### Issue: Slow Performance

**Possible Causes:**
1. Batch size too large for network/memory
2. No proper indexing on target tables
3. Extracting during peak hours

**Solutions:**
- Reduce `@RowCount` parameter
- Add indexes on PagingId, LastUpdated, TenantId in staging tables
- Schedule during off-peak hours
- Use parallel extraction for independent entities

### Issue: Missing Records

**Possible Causes:**
1. Not handling soft deletes properly
2. Clock skew between systems
3. Overlapping extraction windows

**Solutions:**
- Include `IsDeleted = 1` records in extraction
- Add buffer to date ranges (e.g., subtract 5 minutes from `@LastUpdatedStart`)
- Use PagingId-based extraction

### Issue: Duplicate Records

**Possible Causes:**
1. Retry logic without proper deduplication
2. Concurrent extractions
3. Not tracking PagingId properly

**Solutions:**
- Implement upsert logic in target (MERGE statements)
- Use orchestration framework to prevent concurrent runs
- Store and increment PagingId properly

---

## Appendix A: Data Type Mappings

Common SQL Server to target system data type mappings:

| SQL Server Type | Description | PostgreSQL | MySQL | Snowflake | Spark SQL |
|-----------------|-------------|------------|-------|-----------|-----------|
| `uniqueidentifier` | GUID | `uuid` | `char(36)` | `varchar(36)` | `string` |
| `rowversion` | 8-byte incrementing | `bytea` | `binary(8)` | `binary(8)` | `binary` |
| `datetime` | Date + time | `timestamp` | `datetime` | `timestamp` | `timestamp` |
| `nvarchar(max)` | Unicode text | `text` | `text` | `varchar` | `string` |
| `bit` | Boolean | `boolean` | `tinyint(1)` | `boolean` | `boolean` |

---

## Appendix B: Glossary

| Term | Definition |
|------|------------|
| **Assignment** | Core work order entity in Obzervr representing a task or job |
| **Assignment Point** | A specific location or asset where work is performed |
| **Field Measurement** | Data collected during field work (readings, observations, answers to questions in forms) |
| **Soft Delete** | Logical deletion where `IsDeleted = 1`, record remains in database |
| **Snapshot** | Point-in-time copy of data, typically for historical analysis |
| **Template Group** | Collection of work templates grouped for organizational purposes |
| **Work Template** | Reusable definition for assignments with predefined structure |
| **Time Series** | Sequential measurements over time, typically sensor data |

---

## Appendix C: Support & Contact

For technical support regarding the Data Pipeline:

- **Documentation**: [Obzervr Data Pipeline Integration Guide](./Obzervr-Data-Pipeline-Integration.ipynb)
- **Rate Limiting Issues**: Contact your Obzervr administrator for lease configuration
- **Bug Reports**: Submit through your organization's support channel
- **Feature Requests**: Discuss with your Obzervr account manager

---

## Document Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-07-12 | System | Initial specification document |

---