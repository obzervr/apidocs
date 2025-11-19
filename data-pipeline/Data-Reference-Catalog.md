# Obzervr Data Pipeline - Data Reference Catalog

---

## Table of Contents

- [Overview](#overview)
- [Quick Reference Table](#quick-reference-table)
- [Stored Procedure Details](#stored-procedure-details)

## Overview

This catalog provides comprehensive documentation for all stored procedures available in the Obzervr Data Pipeline. Each stored procedure is classified by data volume and includes detailed information about:

- Input parameters and usage examples
- Paging and incremental load capabilities
- Complete result set schema with column definitions
- Data characteristics and usage notes

### Classification Levels

- **Extra Large**: High-volume tables requiring careful paging strategy (1M+ rows)
- **Large**: Significant data volumes, paging recommended (100K rows)
- **Medium**: Moderate data volumes, may vary by customer (10K rows)
- **Small**: Low-volume dimension tables and lookups (<1K rows)

---


## Quick Reference Table

| Table Name | Classification | Stored Procedure | Max Page Size | Paging Required | Soft Delete |
|------------|----------------|------------------|---------------|-----------------|-------------|
| [FactAssignmentDetailsSnapshot](#factassignmentdetailssnapshot) | Extra Large | `pipeline.sp_GetAssignmentDetailsSnapshot` | 100,000 | Yes | No |
| [FactAssignmentProgressSnapshot](#factassignmentprogresssnapshot) | Extra Large | `pipeline.sp_GetAssignmentProgressSnapshot` | 100,000 | Yes | No |
| [FactAuditCommandCountSnapshotHourly](#factauditcommandcountsnapshothourly) | Extra Large | `pipeline.sp_GetAuditCommandCountSnapshotHourly` | 100,000 | Yes | No |
| [FactAuditsUserAssignment](#factauditsuserassignment) | Extra Large | `pipeline.sp_GetAuditsUserAssignment` | 100,000 | Yes | No |
| [FactTimeSeries](#facttimeseries) | Extra Large | `pipeline.sp_GetTimeSeries` | 100,000 | Yes | Yes |
| [FactTimeSeriesFieldMeasurements](#facttimeseriesfieldmeasurements) | Extra Large | `pipeline.sp_GetTimeSeriesFieldMeasurements` | 1,000,000 | Yes | Yes |
| [DimAssignments](#dimassignments) | Large | `pipeline.sp_GetAssignments` | 100,000 | Yes | Yes |
| [FactAssignmentDeclinedReasons](#factassignmentdeclinedreasons) | Large | `pipeline.sp_GetAssignmentDeclinedReasons` | 100,000 | Yes | No |
| [FactAssignmentExceptions](#factassignmentexceptions) | Large | `pipeline.sp_GetAssignmentExceptions` | 100,000 | Yes | No |
| [FactAssignmentFieldOperators](#factassignmentfieldoperators) | Large | `pipeline.sp_GetAssignmentFieldOperators` | 100,000 | Yes | Yes |
| [FactAssignmentTags](#factassignmenttags) | Large | `pipeline.sp_GetAssignmentTags` | 100,000 | Yes | Yes |
| [DimAssignmentPoints](#dimassignmentpoints) | Medium | `pipeline.sp_GetAssignmentPoints` | 100,000 | Yes | Yes |
| [FactAssignmentPointAttributes](#factassignmentpointattributes) | Medium | `pipeline.sp_GetAssignmentPointAttributes` | 100,000 | Yes | Yes |
| [FactTimeSeriesFieldMeasurementDocuments](#facttimeseriesfieldmeasurementdocuments) | Medium | `pipeline.sp_GetTimeSeriesFieldMeasurementDocuments` | 100,000 | Yes | No |
| [DimAssignmentCategories](#dimassignmentcategories) | Small | `pipeline.sp_GetAssignmentCategories` | 100 | No | Yes |
| [DimAssignmentStatus](#dimassignmentstatus) | Small | `pipeline.sp_GetAssignmentStatus` | 10 | No | No |
| [DimFieldMeasurementTables](#dimfieldmeasurementtables) | Small | `pipeline.sp_GetFieldMeasurementTableDefinitions` | 100 | Yes | Yes |
| [DimSites](#dimsites) | Small | `pipeline.sp_GetSites` | 100 | No | Yes |
| [DimSubSites](#dimsubsites) | Small | `pipeline.sp_GetSubSites` | 100 | No | Yes |
| [DimTeams](#dimteams) | Small | `pipeline.sp_GetTeams` | 100 | Yes | Yes |
| [DimTemplateGroups](#dimtemplategroups) | Small | `pipeline.sp_GetTemplateGroups` | 100 | No | Yes |
| [DimUsers](#dimusers) | Small | `pipeline.sp_GetUsers` | 1,000 | Yes | Yes |
| [DimWorkTemplates](#dimworktemplates) | Small | `pipeline.sp_GetWorkTemplates` | 1,000 | Yes | Yes |
| [FactFieldMeasurementTables](#factfieldmeasurementtables) | Small | `pipeline.sp_GetFieldMeasurementTableContents` | 10,000 | Yes | Yes |
| [FactTeamUsers](#factteamusers) | Small | `pipeline.sp_GetTeamUsers` | 1,000 | Yes | Yes |
| [FactTemplateGroupWorkTemplates](#facttemplategroupworktemplates) | Small | `pipeline.sp_GetTemplateGroupTemplates` | 100 | Yes | No |
| [FactTimeSeriesSectionDocuments](#facttimeseriessectiondocuments) | Small | `pipeline.sp_GetTimeSeriesSectionDocuments` | 10,000 | Yes | No |
| [LabelAlias](#labelalias) | Small | `pipeline.sp_GetLabelAlias` | 100 | No | Yes |
| [TenantSettings](#tenantsettings) | Small | `pipeline.sp_GetTenants` | 10 | No | No |

---


## Stored Procedure Details


### FactAssignmentDetailsSnapshot

**Stored Procedure:** `pipeline.sp_GetAssignmentDetailsSnapshot`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Extra Large |
| Default Page Size | N/A |
| Max Page Size | 100,000 |
| Paging (Incremental) | Yes |
| Paging (Historical) | Yes |
| Soft Delete Supported | No |
| Insert Only | Yes |

#### Description

Retrieves point-in-time snapshots of assignment details showing status changes,
team assignments, and user assignments over time. Use for tracking assignment
lifecycle, historical analysis, and understanding how assignments evolve.
Supports temporal analysis of assignment ownership and status transitions.

#### Usage Example

```sql
exec pipeline.sp_GetAssignmentDetailsSnapshot
@LastPagingId = 0x0000000000003030
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_FactAssignmentDetailsSnapshot]
(
    [TenantId] uniqueidentifier NULL,
    [AssignmentId] uniqueidentifier NULL,
    [UserId] uniqueidentifier NULL,
    [TeamId] uniqueidentifier NULL,
    [Status] int NULL,
    [AssignedTo] uniqueidentifier NULL,
    [FromDate] datetimeoffset(7) NULL,
    [ToDate] datetimeoffset(7) NULL,
    [RequiredDate] datetimeoffset(7) NULL,
    [SnapshotTimestamp] datetimeoffset(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [LastLoaded] datetime2(7) NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| AssignmentId | uniqueidentifier | Links to the parent assignment record being tracked |
| UserId | uniqueidentifier | User assigned to this assignment at the time of snapshot |
| TeamId | uniqueidentifier | Team responsible for this assignment at the time of snapshot |
| Status | int | Assignment status at snapshot time (e.g., Open, In Progress, Completed, Closed) |
| AssignedTo | uniqueidentifier | Display name or identifier of the assigned entity (user or team) |
| FromDate | datetimeoffset(7) | Start date/time when this assignment configuration became active |
| ToDate | datetimeoffset(7) | End date/time when this assignment configuration ended (NULL if current) |
| RequiredDate | datetimeoffset(7) | Due date or required completion date for the assignment |
| SnapshotTimestamp | datetimeoffset(7) | Date/time when this snapshot record was captured |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| LastLoaded | datetime2(7) | Timestamp of last modification, used for incremental loading |

---


### FactAssignmentProgressSnapshot

**Stored Procedure:** `pipeline.sp_GetAssignmentProgressSnapshot`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Extra Large |
| Default Page Size | N/A |
| Max Page Size | 100,000 |
| Paging (Incremental) | Yes |
| Paging (Historical) | Yes |
| Soft Delete Supported | No |
| Insert Only | Yes |

#### Description

Captures assignment progress metrics at specific points in time, tracking completion
percentages across planning, work, and overall completion dimensions. Essential for
monitoring project progress, velocity tracking, and identifying bottlenecks in workflow.

#### Usage Example

```sql
exec pipeline.sp_GetAssignmentProgressSnapshot
@LastPagingId = 0x00000000002C7DF7
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_FactAssignmentProgressSnapshot]
(
    [TenantId] uniqueidentifier NULL,
    [AssignmentId] uniqueidentifier NULL,
    [TotalPercent] numeric(30,5) NULL,
    [PlanPercent] numeric(30,5) NULL,
    [WorkPercent] numeric(30,5) NULL,
    [CompletePercent] numeric(30,5) NULL,
    [Total] int NULL,
    [Complete] int NULL,
    [LastSyncTimestamp] datetime NULL,
    [SnapshotTimestamp] datetime NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [LastLoaded] datetime2(7) NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| AssignmentId | uniqueidentifier | Links to the assignment being measured |
| TotalPercent | numeric(30,5) | Overall completion percentage (0-100) across all dimensions |
| PlanPercent | numeric(30,5) | Percentage of planning phase completed |
| WorkPercent | numeric(30,5) | Percentage of work/execution phase completed |
| CompletePercent | numeric(30,5) | Percentage of completion/closeout phase completed |
| Total | int | Total count or measure of assignment items |
| Complete | int | Count or measure of completed assignment items |
| LastSyncTimestamp | datetime | Timestamp when data was last synchronized from source system |
| SnapshotTimestamp | datetime | Date/time when this progress snapshot was captured |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| LastLoaded | datetime2(7) | Timestamp of last modification, used for incremental loading |

---


### FactAuditCommandCountSnapshotHourly

**Stored Procedure:** `pipeline.sp_GetAuditCommandCountSnapshotHourly`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Extra Large |
| Default Page Size | N/A |
| Max Page Size | 100,000 |
| Paging (Incremental) | Yes |
| Paging (Historical) | Yes |
| Soft Delete Supported | No |
| Insert Only | Yes |

#### Description

Provides hourly aggregated counts of commands executed by users, enabling usage analytics,
performance monitoring, and user activity tracking. Use for understanding system utilization
patterns, identifying peak usage times, and user engagement analysis.

#### Usage Example

```sql
exec pipeline.sp_GetAuditCommandCountSnapshotHourly
@LastPagingId = 0x00000000002EE0CB
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_FactAuditCommandCountSnapshotHourly]
(
    [TenantId] uniqueidentifier NULL,
    [UserId] uniqueidentifier NULL,
    [CommandName] nvarchar(200) NULL,
    [Count] int NULL,
    [SnapshotTimestamp] datetimeoffset(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [LastLoaded] datetime2(7) NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| UserId | uniqueidentifier | User who executed the commands |
| CommandName | nvarchar(200) | Name or identifier of the command/action executed |
| Count | int | Number of times the command was executed in the snapshot period |
| SnapshotTimestamp | datetimeoffset(7) | Hourly timestamp marking when this aggregation was captured |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| LastLoaded | datetime2(7) | Timestamp of last modification, used for incremental loading |

---


### FactAuditsUserAssignment

**Stored Procedure:** `pipeline.sp_GetAuditsUserAssignment`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Extra Large |
| Default Page Size | N/A |
| Max Page Size | 100,000 |
| Paging (Incremental) | Yes |
| Paging (Historical) | Yes |
| Soft Delete Supported | No |
| Insert Only | Yes |

#### Description

Records detailed audit trail of user actions on assignments, capturing who did what and when.
Critical for compliance, troubleshooting, security analysis, and understanding user behavior
patterns. Enables reconstruction of assignment lifecycle events.

#### Usage Example

```sql
exec [pipeline].sp_GetAuditsUserAssignment
@LastPagingId = 0x00000000001FA570
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_FactAuditsUserAssignment]
(
    [Id] uniqueidentifier NULL,
    [TenantId] uniqueidentifier NULL,
    [UserId] uniqueidentifier NULL,
    [CommandName] nvarchar(200) NULL,
    [AssignmentId] uniqueidentifier NULL,
    [FiredTimestamp] datetime NULL,
    [LastUpdated] datetime2(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this audit record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| UserId | uniqueidentifier | User who performed the audited action |
| CommandName | nvarchar(200) | Name of the command or action that was audited |
| AssignmentId | uniqueidentifier | Assignment that was affected by the action |
| FiredTimestamp | datetime | Exact date/time when the audited event occurred |
| LastUpdated | datetime2(7) | Timestamp of last modification to this audit record |
| PagingId | rowversion | Rowversion used for paging through large result sets |

---


### FactTimeSeries

**Stored Procedure:** `pipeline.sp_GetTimeSeries`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Extra Large |
| Default Page Size | N/A |
| Max Page Size | 100,000 |
| Paging (Incremental) | Yes |
| Paging (Historical) | Yes |
| Soft Delete Supported | Yes |
| Insert Only | No |

#### Description

Retrieves FactTimeSeries data. Add detailed description here.

#### Usage Example

```sql
exec pipeline.sp_GetTimeSeries
@LastPagingId =0x0000000000003030
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_FactTimeSeries]
(
    [Id] uniqueidentifier NULL,
    [TenantId] uniqueidentifier NULL,
    [AssignmentId] uniqueidentifier NULL,
    [LastOperatedAt] datetime NULL,
    [TimeSeriesTypeInstanceId] uniqueidentifier NULL,
    [SeriesInstanceName] nvarchar(MAX) NULL,
    [SeriesName] nvarchar(MAX) NULL,
    [SeriesIdentifier] nvarchar(500) NULL,
    [SequenceNumber] int NULL,
    [GroupFragmentReference] uniqueidentifier NULL,
    [CompletedAt] datetime NULL,
    [CompletedBy] uniqueidentifier NULL,
    [ParentId] uniqueidentifier NULL,
    [CreatedDate] datetimeoffset(7) NULL,
    [LastUpdated] datetimeoffset(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| AssignmentId | uniqueidentifier | Links to the Assignment record |
| LastOperatedAt | datetime | Date/time timestamp |
| TimeSeriesTypeInstanceId | uniqueidentifier | Links to the TimeSeriesTypeInstance record |
| SeriesInstanceName | nvarchar(MAX) | Data field |
| SeriesName | nvarchar(MAX) | Data field |
| SeriesIdentifier | nvarchar(500) | Data field |
| SequenceNumber | int | Data field |
| GroupFragmentReference | uniqueidentifier | Data field |
| CompletedAt | datetime | Date/time timestamp |
| CompletedBy | uniqueidentifier | Data field |
| ParentId | uniqueidentifier | Links to the Parent record |
| CreatedDate | datetimeoffset(7) | Date/time when this record was created |
| LastUpdated | datetimeoffset(7) | Timestamp of last modification, used for incremental loading |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |

---


### FactTimeSeriesFieldMeasurements

**Stored Procedure:** `pipeline.sp_GetTimeSeriesFieldMeasurements`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Extra Large |
| Default Page Size | N/A |
| Max Page Size | 1,000,000 |
| Paging (Incremental) | Yes |
| Paging (Historical) | Yes |
| Soft Delete Supported | Yes |
| Insert Only | No |

#### Description

Retrieves FactTimeSeriesFieldMeasurements data. Add detailed description here.

#### Usage Example

```sql
exec pipeline.sp_GetTimeSeriesFieldMeasurements
@LastPagingId = 0x0000000000003030
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_FactTimeSeriesFieldMeasurements]
(
    [Id] uniqueidentifier NULL,
    [TenantId] uniqueidentifier NULL,
    [ATSFMId] uniqueidentifier NULL,
    [TimeSeriesId] uniqueidentifier NULL,
    [Comments] nvarchar(MAX) NULL,
    [CapturedOn] datetime NULL,
    [CompletedAt] datetime NULL,
    [CapturedBy] uniqueidentifier NULL,
    [CompletedBy] uniqueidentifier NULL,
    [FieldMeasurementName] nvarchar(250) NULL,
    [FieldMeasurementIdentifier] nvarchar(250) NULL,
    [SectionName] nvarchar(250) NULL,
    [SectionIdentifier] nvarchar(250) NULL,
    [DataType] int NULL,
    [AssignmentId] uniqueidentifier NULL,
    [LowerBoundary] real NULL,
    [UpperBoundary] real NULL,
    [Preface] nvarchar(MAX) NULL,
    [Postface] nvarchar(MAX) NULL,
    [Unit] nvarchar(100) NULL,
    [Reading] nvarchar(MAX) NULL,
    [SelectedMultiSelectValue] nvarchar(MAX) NULL,
    [CreatedDate] datetimeoffset(7) NULL,
    [LastUpdated] datetimeoffset(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| ATSFMId | uniqueidentifier | Links to the ATSFM record |
| TimeSeriesId | uniqueidentifier | Links to the TimeSeries record |
| Comments | nvarchar(MAX) | Additional notes or comments |
| CapturedOn | datetime | Date/time timestamp |
| CompletedAt | datetime | Date/time timestamp |
| CapturedBy | uniqueidentifier | Data field |
| CompletedBy | uniqueidentifier | Data field |
| FieldMeasurementName | nvarchar(250) | Data field |
| FieldMeasurementIdentifier | nvarchar(250) | Data field |
| SectionName | nvarchar(250) | Data field |
| SectionIdentifier | nvarchar(250) | Data field |
| DataType | int | Data field |
| AssignmentId | uniqueidentifier | Links to the Assignment record |
| LowerBoundary | real | Data field |
| UpperBoundary | real | Data field |
| Preface | nvarchar(MAX) | Data field |
| Postface | nvarchar(MAX) | Data field |
| Unit | nvarchar(100) | Data field |
| Reading | nvarchar(MAX) | Data field |
| SelectedMultiSelectValue | nvarchar(MAX) | Data field |
| CreatedDate | datetimeoffset(7) | Date/time when this record was created |
| LastUpdated | datetimeoffset(7) | Timestamp of last modification, used for incremental loading |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |

---


### DimAssignments

**Stored Procedure:** `pipeline.sp_GetAssignments`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Large |
| Default Page Size | N/A |
| Max Page Size | 100,000 |
| Paging (Incremental) | Yes |
| Paging (Historical) | Yes |
| Soft Delete Supported | Yes |
| Insert Only | No |

#### Description

Retrieves DimAssignments data. Add detailed description here.

#### Usage Example

```sql
exec pipeline.sp_GetAssignments
@LastPagingId = 0x0000000000003030
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_DimAssignments]
(
    [TenantId] uniqueidentifier NULL,
    [Id] uniqueidentifier NULL,
    [AssignmentCode] nvarchar(50) NULL,
    [AssignmentPointId] uniqueidentifier NULL,
    [AssignedTo] uniqueidentifier NULL,
    [FromDate] datetime2(7) NULL,
    [ToDate] datetime2(7) NULL,
    [Status] int NULL,
    [CreatedBy] uniqueidentifier NULL,
    [WorkTemplateId] uniqueidentifier NULL,
    [TeamId] uniqueidentifier NULL,
    [CompletedBy] uniqueidentifier NULL,
    [FinalisedBy] uniqueidentifier NULL,
    [CompletedOn] datetime2(7) NULL,
    [FinalisedOn] datetime2(7) NULL,
    [CancelledOn] datetime2(7) NULL,
    [DeclinedOn] datetime2(7) NULL,
    [RequiredDate] datetime2(7) NULL,
    [AssignmentCategoryId] uniqueidentifier NULL,
    [AssignmentTitle] nvarchar(200) NULL,
    [Revision] nvarchar(25) NULL,
    [Effort] bigint NULL,
    [CreatedDate] datetimeoffset(7) NULL,
    [LastUpdated] datetimeoffset(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| Id | uniqueidentifier | Unique identifier for this record |
| AssignmentCode | nvarchar(50) | Data field |
| AssignmentPointId | uniqueidentifier | Links to the AssignmentPoint record |
| AssignedTo | uniqueidentifier | Data field |
| FromDate | datetime2(7) | Date/time timestamp |
| ToDate | datetime2(7) | Date/time timestamp |
| Status | int | Current status of the record |
| CreatedBy | uniqueidentifier | User who created this record |
| WorkTemplateId | uniqueidentifier | Links to the WorkTemplate record |
| TeamId | uniqueidentifier | Links to the Team record |
| CompletedBy | uniqueidentifier | Data field |
| FinalisedBy | uniqueidentifier | Data field |
| CompletedOn | datetime2(7) | Date/time timestamp |
| FinalisedOn | datetime2(7) | Date/time timestamp |
| CancelledOn | datetime2(7) | Date/time timestamp |
| DeclinedOn | datetime2(7) | Date/time timestamp |
| RequiredDate | datetime2(7) | Date/time timestamp |
| AssignmentCategoryId | uniqueidentifier | Links to the AssignmentCategory record |
| AssignmentTitle | nvarchar(200) | Data field |
| Revision | nvarchar(25) | Data field |
| Effort | bigint | Data field |
| CreatedDate | datetimeoffset(7) | Date/time when this record was created |
| LastUpdated | datetimeoffset(7) | Timestamp of last modification, used for incremental loading |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |

---


### FactAssignmentDeclinedReasons

**Stored Procedure:** `pipeline.sp_GetAssignmentDeclinedReasons`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Large |
| Default Page Size | N/A |
| Max Page Size | 100,000 |
| Paging (Incremental) | Yes |
| Paging (Historical) | Yes |
| Soft Delete Supported | No |
| Insert Only | No |

#### Description

Retrieves FactAssignmentDeclinedReasons data. Add detailed description here.

#### Usage Example

```sql
exec pipeline.sp_GetAssignmentDeclinedReasons
@LastPagingId = 0x0000000000003030
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_FactAssignmentDeclinedReasons]
(
    [Id] uniqueidentifier NULL,
    [TenantId] uniqueidentifier NULL,
    [AssignmentId] uniqueidentifier NULL,
    [DeclinedBy] uniqueidentifier NULL,
    [DeclinedOn] datetime NULL,
    [DeclinedByFullName] nvarchar(250) NULL,
    [ReasonForDecliningComment] nvarchar(500) NULL,
    [ReasonForDeclining] nvarchar(MAX) NULL,
    [CreatedDate] datetimeoffset(7) NULL,
    [LastUpdated] datetimeoffset(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| AssignmentId | uniqueidentifier | Links to the Assignment record |
| DeclinedBy | uniqueidentifier | Data field |
| DeclinedOn | datetime | Date/time timestamp |
| DeclinedByFullName | nvarchar(250) | Data field |
| ReasonForDecliningComment | nvarchar(500) | Data field |
| ReasonForDeclining | nvarchar(MAX) | Data field |
| CreatedDate | datetimeoffset(7) | Date/time when this record was created |
| LastUpdated | datetimeoffset(7) | Timestamp of last modification, used for incremental loading |
| PagingId | rowversion | Rowversion used for paging through large result sets |

---


### FactAssignmentExceptions

**Stored Procedure:** `pipeline.sp_GetAssignmentExceptions`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Large |
| Default Page Size | N/A |
| Max Page Size | 100,000 |
| Paging (Incremental) | Yes |
| Paging (Historical) | Yes |
| Soft Delete Supported | No |
| Insert Only | No |

#### Description

Retrieves FactAssignmentExceptions data. Add detailed description here.

#### Usage Example

```sql
exec pipeline.sp_GetAssignmentExceptions
@LastPagingId = 0x0000000000003030
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_FactAssignmentExceptions]
(
    [Id] uniqueidentifier NULL,
    [TenantId] uniqueidentifier NULL,
    [AssignmentId] uniqueidentifier NULL,
    [FieldMeasureId] uniqueidentifier NULL,
    [Priority] int NULL,
    [RaisedAt] datetime NULL,
    [RaisedBy] uniqueidentifier NULL,
    [RaisedByName] nvarchar(250) NULL,
    [ResolvedAt] datetime NULL,
    [ResolvedBy] uniqueidentifier NULL,
    [ResolvedByName] nvarchar(250) NULL,
    [Comment] nvarchar(MAX) NULL,
    [CreatedDate] datetimeoffset(7) NULL,
    [LastUpdated] datetimeoffset(7) NULL,
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| AssignmentId | uniqueidentifier | Links to the Assignment record |
| FieldMeasureId | uniqueidentifier | Links to the FieldMeasure record |
| Priority | int | Data field |
| RaisedAt | datetime | Date/time timestamp |
| RaisedBy | uniqueidentifier | Data field |
| RaisedByName | nvarchar(250) | Data field |
| ResolvedAt | datetime | Date/time timestamp |
| ResolvedBy | uniqueidentifier | Data field |
| ResolvedByName | nvarchar(250) | Data field |
| Comment | nvarchar(MAX) | Additional notes or comments |
| CreatedDate | datetimeoffset(7) | Date/time when this record was created |
| LastUpdated | datetimeoffset(7) | Timestamp of last modification, used for incremental loading |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |

---


### FactAssignmentFieldOperators

**Stored Procedure:** `pipeline.sp_GetAssignmentFieldOperators`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Large |
| Default Page Size | N/A |
| Max Page Size | 100,000 |
| Paging (Incremental) | Yes |
| Paging (Historical) | Yes |
| Soft Delete Supported | Yes |
| Insert Only | No |

#### Description

Retrieves FactAssignmentFieldOperators data. Add detailed description here.

#### Usage Example

```sql
exec pipeline.sp_GetAssignmentFieldOperators
@LastPagingId = 0x0000000000003030
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_FactAssignmentFieldOperators]
(
    [Id] uniqueidentifier NULL,
    [TenantId] uniqueidentifier NULL,
    [AssignmentId] uniqueidentifier NULL,
    [FieldOperatorId] uniqueidentifier NULL,
    [CreatedDate] datetimeoffset(7) NULL,
    [LastUpdated] datetimeoffset(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| AssignmentId | uniqueidentifier | Links to the Assignment record |
| FieldOperatorId | uniqueidentifier | Links to the FieldOperator record |
| CreatedDate | datetimeoffset(7) | Date/time when this record was created |
| LastUpdated | datetimeoffset(7) | Timestamp of last modification, used for incremental loading |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |

---


### FactAssignmentTags

**Stored Procedure:** `pipeline.sp_GetAssignmentTags`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Large |
| Default Page Size | N/A |
| Max Page Size | 100,000 |
| Paging (Incremental) | Yes |
| Paging (Historical) | Yes |
| Soft Delete Supported | Yes |
| Insert Only | No |

#### Description

Retrieves FactAssignmentTags data. Add detailed description here.

#### Usage Example

```sql
exec pipeline.sp_GetAssignmentTags
@LastPagingId = 0x0000000000003030
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_FactAssignmentTags]
(
    [Id] uniqueidentifier NULL,
    [TenantId] uniqueidentifier NULL,
    [AssignmentId] uniqueidentifier NULL,
    [Key] nvarchar(250) NULL,
    [Value] nvarchar(500) NULL,
    [IsVisible] bit NULL,
    [IsInteractive] bit NULL,
    [CreatedDate] datetimeoffset(7) NULL,
    [LastUpdated] datetimeoffset(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| AssignmentId | uniqueidentifier | Links to the Assignment record |
| Key | nvarchar(250) | Data field |
| Value | nvarchar(500) | Data field |
| IsVisible | bit | Indicates if visible condition is true |
| IsInteractive | bit | Indicates if interactive condition is true |
| CreatedDate | datetimeoffset(7) | Date/time when this record was created |
| LastUpdated | datetimeoffset(7) | Timestamp of last modification, used for incremental loading |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |

---


### DimAssignmentPoints

**Stored Procedure:** `pipeline.sp_GetAssignmentPoints`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Medium |
| Default Page Size | N/A |
| Max Page Size | 100,000 |
| Paging (Incremental) | Yes |
| Paging (Historical) | Yes |
| Soft Delete Supported | Yes |
| Insert Only | No |

#### Description

Retrieves DimAssignmentPoints data. Add detailed description here.

**Notes:** It is Large to Extra Large based on customers. Bluescope has Extra large data

#### Usage Example

```sql
exec pipeline.sp_GetAssignmentPoints
@LastPagingId = 0x0000000000003030
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_DimAssignmentPoints]
(
    [Id] uniqueidentifier NULL,
    [TenantId] uniqueidentifier NULL,
    [PointId] nvarchar(50) NULL,
    [PointName] nvarchar(100) NULL,
    [ParentId] uniqueidentifier NULL,
    [SubsiteId] uniqueidentifier NULL,
    [AssignmentPointTypeName] nvarchar(500) NULL,
    [APLatitude] numeric(10,7) NULL,
    [APLongitude] numeric(10,7) NULL,
    [CreatedDate] datetimeoffset(7) NULL,
    [LastUpdated] datetimeoffset(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| PointId | nvarchar(50) | Links to the Point record |
| PointName | nvarchar(100) | Data field |
| ParentId | uniqueidentifier | Links to the Parent record |
| SubsiteId | uniqueidentifier | Links to the Subsite record |
| AssignmentPointTypeName | nvarchar(500) | Data field |
| APLatitude | numeric(10,7) | Data field |
| APLongitude | numeric(10,7) | Data field |
| CreatedDate | datetimeoffset(7) | Date/time when this record was created |
| LastUpdated | datetimeoffset(7) | Timestamp of last modification, used for incremental loading |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |

---


### FactAssignmentPointAttributes

**Stored Procedure:** `pipeline.sp_GetAssignmentPointAttributes`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Medium |
| Default Page Size | N/A |
| Max Page Size | 100,000 |
| Paging (Incremental) | Yes |
| Paging (Historical) | Yes |
| Soft Delete Supported | Yes |
| Insert Only | No |

#### Description

Retrieves FactAssignmentPointAttributes data. Add detailed description here.

**Notes:** It is Large to Extra Large based on customers. Bluescope has Extra large data

#### Usage Example

```sql
exec pipeline.sp_GetAssignmentPointAttributes
@LastPagingId = 0x0000000000003030
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_FactAssignmentPointAttributes]
(
    [Id] uniqueidentifier NULL,
    [TenantId] uniqueidentifier NULL,
    [AttributeName] nvarchar(250) NULL,
    [AttributeValue] nvarchar(MAX) NULL,
    [AttributeGroupName] nvarchar(250) NULL,
    [AssignmentPointId] uniqueidentifier NULL,
    [CreatedDate] datetimeoffset(7) NULL,
    [LastUpdated] datetimeoffset(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| AttributeName | nvarchar(250) | Data field |
| AttributeValue | nvarchar(MAX) | Data field |
| AttributeGroupName | nvarchar(250) | Data field |
| AssignmentPointId | uniqueidentifier | Links to the AssignmentPoint record |
| CreatedDate | datetimeoffset(7) | Date/time when this record was created |
| LastUpdated | datetimeoffset(7) | Timestamp of last modification, used for incremental loading |
| PagingId | rowversion | Rowversion used for paging through large result sets |

---


### FactTimeSeriesFieldMeasurementDocuments

**Stored Procedure:** `pipeline.sp_GetTimeSeriesFieldMeasurementDocuments`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Medium |
| Default Page Size | N/A |
| Max Page Size | 100,000 |
| Paging (Incremental) | Yes |
| Paging (Historical) | Yes |
| Soft Delete Supported | No |
| Insert Only | No |

#### Description

Retrieves FactTimeSeriesFieldMeasurementDocuments data. Add detailed description here.

#### Usage Example

```sql
exec pipeline.sp_GetTimeSeriesFieldMeasurementDocuments
@LastPagingId = 0x0000000000003030
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_FactTimeSeriesFieldMeasurementDocuments]
(
    [DocumentId] uniqueidentifier NULL,
    [TenantId] uniqueidentifier NULL,
    [TimeSeriesFieldMeasurementID] uniqueidentifier NULL,
    [AssignmentTimeSeriesFieldMeasurementId] uniqueidentifier NULL,
    [ParentId] uniqueidentifier NULL,
    [FileName] nvarchar(500) NULL,
    [DocumentURL] nvarchar(500) NULL,
    [CreatedDate] datetimeoffset(7) NULL,
    [LastUpdated] datetimeoffset(7) NULL,
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| DocumentId | uniqueidentifier | Links to the Document record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| TimeSeriesFieldMeasurementID | uniqueidentifier | Data field |
| AssignmentTimeSeriesFieldMeasurementId | uniqueidentifier | Links to the AssignmentTimeSeriesFieldMeasurement record |
| ParentId | uniqueidentifier | Links to the Parent record |
| FileName | nvarchar(500) | Data field |
| DocumentURL | nvarchar(500) | Data field |
| CreatedDate | datetimeoffset(7) | Date/time when this record was created |
| LastUpdated | datetimeoffset(7) | Timestamp of last modification, used for incremental loading |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |

---


### DimAssignmentCategories

**Stored Procedure:** `pipeline.sp_GetAssignmentCategories`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Small |
| Default Page Size | N/A |
| Max Page Size | 100 |
| Paging (Incremental) | No |
| Paging (Historical) | No |
| Soft Delete Supported | Yes |
| Insert Only | No |

#### Description

Retrieves DimAssignmentCategories data. Add detailed description here.

#### Usage Example

```sql
exec pipeline.sp_GetAssignmentCategories
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_DimAssignmentCategories]
(
    [Id] uniqueidentifier NULL,
    [TenantId] uniqueidentifier NULL,
    [Category] nvarchar(250) NULL,
    [Code] nvarchar(2) NULL,
    [Color] nvarchar(7) NULL,
    [Critical] bit NULL,
    [CreatedDate] datetimeoffset(7) NULL,
    [LastUpdated] datetimeoffset(7) NULL,
    [IsDeleted] bit NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| Category | nvarchar(250) | Data field |
| Code | nvarchar(2) | Unique code or identifier |
| Color | nvarchar(7) | Data field |
| Critical | bit | Data field |
| CreatedDate | datetimeoffset(7) | Date/time when this record was created |
| LastUpdated | datetimeoffset(7) | Timestamp of last modification, used for incremental loading |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |
| PagingId | rowversion | Rowversion used for paging through large result sets |

---


### DimAssignmentStatus

**Stored Procedure:** `pipeline.sp_GetAssignmentStatus`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Small |
| Default Page Size | N/A |
| Max Page Size | 10 |
| Paging (Incremental) | No |
| Paging (Historical) | No |
| Soft Delete Supported | No |
| Insert Only | No |

#### Description

Retrieves DimAssignmentStatus data. Add detailed description here.

**Notes:** Lookup table only

#### Usage Example

```sql
exec pipeline.sp_GetAssignmentStatus
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_DimAssignmentStatus]
(
    [Id] int NULL,
    [Name] nvarchar(50) NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | int | Unique identifier for this record |
| Name | nvarchar(50) | Display name |

---


### DimFieldMeasurementTables

**Stored Procedure:** `pipeline.sp_GetFieldMeasurementTableDefinitions`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Small |
| Default Page Size | N/A |
| Max Page Size | 100 |
| Paging (Incremental) | Yes |
| Paging (Historical) | Yes |
| Soft Delete Supported | Yes |
| Insert Only | No |

#### Description

Retrieves DimFieldMeasurementTables data. Add detailed description here.

#### Usage Example

```sql
exec pipeline.sp_GetFieldMeasurementTableDefinitions
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_DimFieldMeasurementTables]
(
    [Id] uniqueidentifier NULL,
    [TenantId] uniqueidentifier NULL,
    [TableName] nvarchar(100) NULL,
    [Column1Name] nvarchar(100) NULL,
    [Column2Name] nvarchar(100) NULL,
    [CreatedDate] datetimeoffset(7) NULL,
    [LastUpdated] datetimeoffset(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| TableName | nvarchar(100) | Data field |
| Column1Name | nvarchar(100) | Data field |
| Column2Name | nvarchar(100) | Data field |
| CreatedDate | datetimeoffset(7) | Date/time when this record was created |
| LastUpdated | datetimeoffset(7) | Timestamp of last modification, used for incremental loading |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |

---


### DimSites

**Stored Procedure:** `pipeline.sp_GetSites`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Small |
| Default Page Size | N/A |
| Max Page Size | 100 |
| Paging (Incremental) | No |
| Paging (Historical) | No |
| Soft Delete Supported | Yes |
| Insert Only | No |

#### Description

Retrieves DimSites data. Add detailed description here.

#### Usage Example

```sql
exec pipeline.sp_GetSites
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_DimSites]
(
    [Id] uniqueidentifier NULL,
    [TenantId] uniqueidentifier NULL,
    [SiteId] nvarchar(50) NULL,
    [SiteName] nvarchar(100) NULL,
    [SiteAddressLine1] nvarchar(500) NULL,
    [SiteAddressLine2] nvarchar(500) NULL,
    [SiteAddressLine3] nvarchar(500) NULL,
    [SiteAddressCity] nvarchar(100) NULL,
    [SiteAddressPostCode] nvarchar(10) NULL,
    [SiteLocationDescription] nvarchar(MAX) NULL,
    [SiteUsageDescription] nvarchar(MAX) NULL,
    [APLatitude] numeric(10,7) NULL,
    [APLongitude] numeric(10,7) NULL,
    [CreatedDate] datetimeoffset(7) NULL,
    [LastUpdated] datetimeoffset(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| SiteId | nvarchar(50) | Links to the Site record |
| SiteName | nvarchar(100) | Data field |
| SiteAddressLine1 | nvarchar(500) | Data field |
| SiteAddressLine2 | nvarchar(500) | Data field |
| SiteAddressLine3 | nvarchar(500) | Data field |
| SiteAddressCity | nvarchar(100) | Data field |
| SiteAddressPostCode | nvarchar(10) | Data field |
| SiteLocationDescription | nvarchar(MAX) | Data field |
| SiteUsageDescription | nvarchar(MAX) | Data field |
| APLatitude | numeric(10,7) | Data field |
| APLongitude | numeric(10,7) | Data field |
| CreatedDate | datetimeoffset(7) | Date/time when this record was created |
| LastUpdated | datetimeoffset(7) | Timestamp of last modification, used for incremental loading |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |

---


### DimSubSites

**Stored Procedure:** `pipeline.sp_GetSubSites`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Small |
| Default Page Size | N/A |
| Max Page Size | 100 |
| Paging (Incremental) | No |
| Paging (Historical) | No |
| Soft Delete Supported | Yes |
| Insert Only | No |

#### Description

Retrieves DimSubSites data. Add detailed description here.

#### Usage Example

```sql
exec pipeline.sp_GetSubSites
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_DimSubSites]
(
    [Id] uniqueidentifier NULL,
    [TenantId] uniqueidentifier NULL,
    [SubSiteId] nvarchar(50) NULL,
    [SubSiteName] nvarchar(100) NULL,
    [SubSiteAddressLine1] nvarchar(500) NULL,
    [SubSiteAddressLine2] nvarchar(500) NULL,
    [SubSiteAddressLine3] nvarchar(500) NULL,
    [SubSiteAddressCity] nvarchar(100) NULL,
    [SubSiteAddressPostCode] nvarchar(10) NULL,
    [SubSiteLocationDescription] nvarchar(MAX) NULL,
    [SubSiteUsageDescription] nvarchar(MAX) NULL,
    [APLatitude] numeric(10,7) NULL,
    [APLongitude] numeric(10,7) NULL,
    [ParentSiteId] uniqueidentifier NULL,
    [CreatedDate] datetimeoffset(7) NULL,
    [LastUpdated] datetimeoffset(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| SubSiteId | nvarchar(50) | Links to the SubSite record |
| SubSiteName | nvarchar(100) | Data field |
| SubSiteAddressLine1 | nvarchar(500) | Data field |
| SubSiteAddressLine2 | nvarchar(500) | Data field |
| SubSiteAddressLine3 | nvarchar(500) | Data field |
| SubSiteAddressCity | nvarchar(100) | Data field |
| SubSiteAddressPostCode | nvarchar(10) | Data field |
| SubSiteLocationDescription | nvarchar(MAX) | Data field |
| SubSiteUsageDescription | nvarchar(MAX) | Data field |
| APLatitude | numeric(10,7) | Data field |
| APLongitude | numeric(10,7) | Data field |
| ParentSiteId | uniqueidentifier | Links to the ParentSite record |
| CreatedDate | datetimeoffset(7) | Date/time when this record was created |
| LastUpdated | datetimeoffset(7) | Timestamp of last modification, used for incremental loading |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |

---


### DimTeams

**Stored Procedure:** `pipeline.sp_GetTeams`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Small |
| Default Page Size | N/A |
| Max Page Size | 100 |
| Paging (Incremental) | Yes |
| Paging (Historical) | Yes |
| Soft Delete Supported | Yes |
| Insert Only | No |

#### Description

Retrieves DimTeams data. Add detailed description here.

#### Usage Example

```sql
exec pipeline.sp_GetTeams
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_DimTeams]
(
    [Id] uniqueidentifier NULL,
    [TenantId] uniqueidentifier NULL,
    [Name] nvarchar(200) NULL,
    [Description] nvarchar(MAX) NULL,
    [CreatedDate] datetimeoffset(7) NULL,
    [LastUpdated] datetimeoffset(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| Name | nvarchar(200) | Display name |
| Description | nvarchar(MAX) | Detailed description or notes |
| CreatedDate | datetimeoffset(7) | Date/time when this record was created |
| LastUpdated | datetimeoffset(7) | Timestamp of last modification, used for incremental loading |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |

---


### DimTemplateGroups

**Stored Procedure:** `pipeline.sp_GetTemplateGroups`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Small |
| Default Page Size | N/A |
| Max Page Size | 100 |
| Paging (Incremental) | No |
| Paging (Historical) | No |
| Soft Delete Supported | Yes |
| Insert Only | No |

#### Description

Retrieves DimTemplateGroups data. Add detailed description here.

#### Usage Example

```sql
exec pipeline.sp_GetTemplateGroupTemplates
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_DimTemplateGroups]
(
    [Id] uniqueidentifier NULL,
    [TenantId] uniqueidentifier NULL,
    [CreatedByUserId] uniqueidentifier NULL,
    [UpdatedByUserId] uniqueidentifier NULL,
    [Name] nvarchar(200) NULL,
    [Identifier] nvarchar(50) NULL,
    [AccessControlled] bit NULL,
    [IsActive] bit NULL,
    [CreatedDate] datetimeoffset(7) NULL,
    [LastUpdated] datetimeoffset(7) NULL,
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| CreatedByUserId | uniqueidentifier | Links to the CreatedByUser record |
| UpdatedByUserId | uniqueidentifier | Links to the UpdatedByUser record |
| Name | nvarchar(200) | Display name |
| Identifier | nvarchar(50) | Unique code or identifier |
| AccessControlled | bit | Data field |
| IsActive | bit | Data field |
| CreatedDate | datetimeoffset(7) | Date/time when this record was created |
| LastUpdated | datetimeoffset(7) | Timestamp of last modification, used for incremental loading |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |

---


### DimUsers

**Stored Procedure:** `pipeline.sp_GetUsers`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Small |
| Default Page Size | N/A |
| Max Page Size | 1,000 |
| Paging (Incremental) | Yes |
| Paging (Historical) | Yes |
| Soft Delete Supported | Yes |
| Insert Only | No |

#### Description

Retrieves DimUsers data. Add detailed description here.

#### Usage Example

```sql
exec pipeline.sp_GetUsers
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_DimUsers]
(
    [TenantId] uniqueidentifier NULL,
    [Id] uniqueidentifier NULL,
    [UserId] uniqueidentifier NULL,
    [Role] nvarchar(MAX) NULL,
    [UserCode] nvarchar(100) NULL,
    [Email] nvarchar(MAX) NULL,
    [FullName] nvarchar(500) NULL,
    [LastSyncTime] datetimeoffset(7) NULL,
    [IsActive] bit NULL,
    [LastUpdated] datetimeoffset(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [ReferenceCode] nvarchar(MAX) NULL,
    [AuthorisationCode] nvarchar(MAX) NULL,
    [Department] nvarchar(MAX) NULL,
    [DepartmentCode] nvarchar(MAX) NULL,
    [Organisation] nvarchar(MAX) NULL,
    [OrganisationCode] nvarchar(MAX) NULL,
    [ContractEffectiveDate] datetimeoffset(7) NULL,
    [ContractEndDate] datetimeoffset(7) NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| Id | uniqueidentifier | Unique identifier for this record |
| UserId | uniqueidentifier | Links to the User record |
| Role | nvarchar(MAX) | Data field |
| UserCode | nvarchar(100) | Data field |
| Email | nvarchar(MAX) | Email address |
| FullName | nvarchar(500) | Complete display name |
| LastSyncTime | datetimeoffset(7) | Last sync timestamp |
| IsActive | bit | Indicates if active condition is true |
| LastUpdated | datetimeoffset(7) | Timestamp of last modification, used for incremental loading |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| ReferenceCode | nvarchar(MAX) | Data field |
| AuthorisationCode | nvarchar(MAX) | Data field |
| Department | nvarchar(MAX) | Data field |
| DepartmentCode | nvarchar(MAX) | Data field |
| Organisation | nvarchar(MAX) | Data field |
| OrganisationCode | nvarchar(MAX) | Data field |
| ContractEffectiveDate | datetimeoffset(7) | Contract effective date |
| ContractEndDate | datetimeoffset(7) | Contract end date |

---


### DimWorkTemplates

**Stored Procedure:** `pipeline.sp_GetWorkTemplates`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Small |
| Default Page Size | N/A |
| Max Page Size | 1,000 |
| Paging (Incremental) | Yes |
| Paging (Historical) | Yes |
| Soft Delete Supported | Yes |
| Insert Only | No |

#### Description

Retrieves DimWorkTemplates data. Add detailed description here.

#### Usage Example

```sql
exec pipeline.sp_GetWorkTemplates
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_DimWorkTemplates]
(
    [Id] uniqueidentifier NULL,
    [TenantId] uniqueidentifier NULL,
    [Identifier] nvarchar(50) NULL,
    [Name] nvarchar(250) NULL,
    [Version] nvarchar(250) NULL,
    [TemplateLink] uniqueidentifier NULL,
    [FragmentType] nvarchar(100) NULL,
    [IsPublished] bit NULL,
    [CreatedDate] datetimeoffset(7) NULL,
    [LastUpdated] datetimeoffset(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [Json] nvarchar(MAX) NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| Identifier | nvarchar(50) | Unique code or identifier |
| Name | nvarchar(250) | Display name |
| Version | nvarchar(250) | Data field |
| TemplateLink | uniqueidentifier | Data field |
| FragmentType | nvarchar(100) | Data field |
| IsPublished | bit | Indicates if published condition is true |
| CreatedDate | datetimeoffset(7) | Date/time when this record was created |
| LastUpdated | datetimeoffset(7) | Timestamp of last modification, used for incremental loading |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| Json | nvarchar(MAX) | Data field |

---


### FactFieldMeasurementTables

**Stored Procedure:** `pipeline.sp_GetFieldMeasurementTableContents`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Small |
| Default Page Size | N/A |
| Max Page Size | 10,000 |
| Paging (Incremental) | Yes |
| Paging (Historical) | Yes |
| Soft Delete Supported | Yes |
| Insert Only | No |

#### Description

Retrieves FactFieldMeasurementTables data. Add detailed description here.

#### Usage Example

```sql
exec pipeline.sp_GetFieldMeasurementTableContents
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_FactFieldMeasurementTables]
(
    [Id] uniqueidentifier NULL,
    [TenantId] uniqueidentifier NULL,
    [FieldMeasurementTableDefinitionId] uniqueidentifier NULL,
    [Column1] nvarchar(500) NULL,
    [Column2] nvarchar(500) NULL,
    [Reference] nvarchar(500) NULL,
    [CreatedDate] datetimeoffset(7) NULL,
    [LastUpdated] datetimeoffset(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| FieldMeasurementTableDefinitionId | uniqueidentifier | Links to the FieldMeasurementTableDefinition record |
| Column1 | nvarchar(500) | Data field |
| Column2 | nvarchar(500) | Data field |
| Reference | nvarchar(500) | Data field |
| CreatedDate | datetimeoffset(7) | Date/time when this record was created |
| LastUpdated | datetimeoffset(7) | Timestamp of last modification, used for incremental loading |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |

---


### FactTeamUsers

**Stored Procedure:** `pipeline.sp_GetTeamUsers`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Small |
| Default Page Size | N/A |
| Max Page Size | 1,000 |
| Paging (Incremental) | Yes |
| Paging (Historical) | Yes |
| Soft Delete Supported | Yes |
| Insert Only | No |

#### Description

Retrieves FactTeamUsers data. Add detailed description here.

#### Usage Example

```sql
exec pipeline.sp_GetTeamUsers
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_FactTeamUsers]
(
    [TenantId] uniqueidentifier NULL,
    [UserId] uniqueidentifier NULL,
    [TeamId] uniqueidentifier NULL,
    [IsSupervisor] bit NULL,
    [IsMember] bit NULL,
    [CreatedDate] datetimeoffset(7) NULL,
    [LastUpdated] datetimeoffset(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| UserId | uniqueidentifier | Links to the User record |
| TeamId | uniqueidentifier | Links to the Team record |
| IsSupervisor | bit | Indicates if supervisor condition is true |
| IsMember | bit | Indicates if member condition is true |
| CreatedDate | datetimeoffset(7) | Date/time when this record was created |
| LastUpdated | datetimeoffset(7) | Timestamp of last modification, used for incremental loading |
| PagingId | rowversion | Rowversion used for paging through large result sets |

---


### FactTemplateGroupWorkTemplates

**Stored Procedure:** `pipeline.sp_GetTemplateGroupTemplates`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Small |
| Default Page Size | N/A |
| Max Page Size | 100 |
| Paging (Incremental) | Yes |
| Paging (Historical) | Yes |
| Soft Delete Supported | No |
| Insert Only | No |

#### Description

Retrieves FactTemplateGroupWorkTemplates data. Add detailed description here.

#### Usage Example

```sql
exec pipeline.sp_GetTemplateGroupTemplates
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_FactTemplateGroupWorkTemplates]
(
    [Id] uniqueidentifier NULL,
    [TenantId] uniqueidentifier NULL,
    [TemplateGroupId] uniqueidentifier NULL,
    [TemplateLink] uniqueidentifier NULL,
    [CreatedDate] datetimeoffset(7) NULL,
    [LastUpdated] datetimeoffset(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| TemplateGroupId | uniqueidentifier | Links to the TemplateGroup record |
| TemplateLink | uniqueidentifier | Data field |
| CreatedDate | datetimeoffset(7) | Date/time when this record was created |
| LastUpdated | datetimeoffset(7) | Timestamp of last modification, used for incremental loading |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |

---


### FactTimeSeriesSectionDocuments

**Stored Procedure:** `pipeline.sp_GetTimeSeriesSectionDocuments`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Small |
| Default Page Size | N/A |
| Max Page Size | 10,000 |
| Paging (Incremental) | Yes |
| Paging (Historical) | Yes |
| Soft Delete Supported | No |
| Insert Only | No |

#### Description

Retrieves FactTimeSeriesSectionDocuments data. Add detailed description here.

#### Usage Example

```sql
exec pipeline.sp_GetTimeSeriesSectionDocuments
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_FactTimeSeriesSectionDocuments]
(
    [DocumentId] uniqueidentifier NULL,
    [TenantId] uniqueidentifier NULL,
    [ParentId] uniqueidentifier NULL,
    [FileName] nvarchar(500) NULL,
    [DocumentKind] int NULL,
    [AssignmentTimeSeriesFieldMeasurementSectionId] uniqueidentifier NULL,
    [TimeSeriesId] uniqueidentifier NULL,
    [DocumentURL] nvarchar(500) NULL,
    [CreatedDate] datetimeoffset(7) NULL,
    [LastUpdated] datetimeoffset(7) NULL,
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| DocumentId | uniqueidentifier | Links to the Document record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| ParentId | uniqueidentifier | Links to the Parent record |
| FileName | nvarchar(500) | Data field |
| DocumentKind | int | Data field |
| AssignmentTimeSeriesFieldMeasurementSectionId | uniqueidentifier | Links to the AssignmentTimeSeriesFieldMeasurementSection record |
| TimeSeriesId | uniqueidentifier | Links to the TimeSeries record |
| DocumentURL | nvarchar(500) | Data field |
| CreatedDate | datetimeoffset(7) | Date/time when this record was created |
| LastUpdated | datetimeoffset(7) | Timestamp of last modification, used for incremental loading |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |

---


### LabelAlias

**Stored Procedure:** `pipeline.sp_GetLabelAlias`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Small |
| Default Page Size | N/A |
| Max Page Size | 100 |
| Paging (Incremental) | No |
| Paging (Historical) | No |
| Soft Delete Supported | Yes |
| Insert Only | No |

#### Description

Retrieves LabelAlias data. Add detailed description here.

**Notes:** Tenant configuration table only

#### Usage Example

```sql
exec pipeline.sp_GetLabelAlias
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_LabelAlias]
(
    [TenantId] uniqueidentifier NULL,
    [Id] uniqueidentifier NULL,
    [LabelKey] nvarchar(200) NULL,
    [LabelValue] nvarchar(200) NULL,
    [LabelPluralValue] nvarchar(200) NULL,
    [CreatedDate] datetimeoffset(7) NULL,
    [LastUpdated] datetimeoffset(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| Id | uniqueidentifier | Unique identifier for this record |
| LabelKey | nvarchar(200) | Data field |
| LabelValue | nvarchar(200) | Data field |
| LabelPluralValue | nvarchar(200) | Data field |
| CreatedDate | datetimeoffset(7) | Date/time when this record was created |
| LastUpdated | datetimeoffset(7) | Timestamp of last modification, used for incremental loading |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |

---


### TenantSettings

**Stored Procedure:** `pipeline.sp_GetTenants`

#### Metadata

| Property | Value |
|----------|-------|
| Classification | Small |
| Default Page Size | N/A |
| Max Page Size | 10 |
| Paging (Incremental) | No |
| Paging (Historical) | No |
| Soft Delete Supported | No |
| Insert Only | No |

#### Description

Retrieves TenantSettings data. Add detailed description here.

#### Usage Example

```sql
exec pipeline.sp_GetTenants
```

#### Staging Table Helper Script

Use this script to create a staging table that matches the stored procedure output.

```sql
CREATE TABLE [dbo].[Staging_TenantSettings]
(
    [TenantId] uniqueidentifier NULL,
    [TenantCode] nvarchar(50) NULL,
    [Timezone] nvarchar(200) NULL,
    [LogoBase64] nvarchar(MAX) NULL,
    [TenantName] nvarchar(100) NULL,
    [TenantURL] nvarchar(100) NULL,
    [PrimaryColour] nvarchar(50) NULL,
    [SecondaryColour] nvarchar(50) NULL,
    [AssignmentPointList1] nvarchar(4000) NULL,
    [AssignmentPointList2] nvarchar(4000) NULL,
    [TempleteList1] nvarchar(4000) NULL,
    [TempleteList2] nvarchar(4000) NULL,
    [SeriesList1] nvarchar(4000) NULL,
    [SeriesList2] nvarchar(4000) NULL,
    [OrganisationKey] nvarchar(300) NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| TenantCode | nvarchar(50) | Data field |
| Timezone | nvarchar(200) | Data field |
| LogoBase64 | nvarchar(MAX) | Data field |
| TenantName | nvarchar(100) | Data field |
| TenantURL | nvarchar(100) | Data field |
| PrimaryColour | nvarchar(50) | Data field |
| SecondaryColour | nvarchar(50) | Data field |
| AssignmentPointList1 | nvarchar(4000) | Data field |
| AssignmentPointList2 | nvarchar(4000) | Data field |
| TempleteList1 | nvarchar(4000) | Data field |
| TempleteList2 | nvarchar(4000) | Data field |
| SeriesList1 | nvarchar(4000) | Data field |
| SeriesList2 | nvarchar(4000) | Data field |
| OrganisationKey | nvarchar(300) | Data field |

---

