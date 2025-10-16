# Obzervr Data Pipeline - Data Reference Catalog

**Version:** 1.0
**Last Updated:** October 12, 2025

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

- **Extra Large**: High-volume tables requiring careful paging strategy (100K+ rows)
- **Large**: Significant data volumes, paging recommended (10K-100K rows)
- **Medium**: Moderate data volumes, may vary by customer (1K-10K rows)
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
    [Status] nvarchar(MAX) NULL,
    [AssignedTo] nvarchar(MAX) NULL,
    [FromDate] datetime2(7) NULL,
    [ToDate] datetime2(7) NULL,
    [RequiredDate] datetime2(7) NULL,
    [SnapshotTimestamp] nvarchar(MAX) NULL,
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
| Status | nvarchar | Assignment status at snapshot time (e.g., Open, In Progress, Completed, Closed) |
| AssignedTo | nvarchar | Display name or identifier of the assigned entity (user or team) |
| FromDate | datetime2 | Start date/time when this assignment configuration became active |
| ToDate | datetime2 | End date/time when this assignment configuration ended (NULL if current) |
| RequiredDate | datetime2 | Due date or required completion date for the assignment |
| SnapshotTimestamp | nvarchar | Date/time when this snapshot record was captured |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| LastLoaded | datetime2 | Timestamp of last modification, used for incremental loading |

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

**Notes:** This snapshot is only for work order complete% purpose, I cannot see any use case for analytics

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
    [TotalPercent] nvarchar(MAX) NULL,
    [PlanPercent] nvarchar(MAX) NULL,
    [WorkPercent] nvarchar(MAX) NULL,
    [CompletePercent] nvarchar(MAX) NULL,
    [Total] nvarchar(MAX) NULL,
    [Complete] nvarchar(MAX) NULL,
    [LastSyncTimestamp] nvarchar(MAX) NULL,
    [SnapshotTimestamp] nvarchar(MAX) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [LastLoaded] datetime2(7) NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| AssignmentId | uniqueidentifier | Links to the assignment being measured |
| TotalPercent | nvarchar | Overall completion percentage (0-100) across all dimensions |
| PlanPercent | nvarchar | Percentage of planning phase completed |
| WorkPercent | nvarchar | Percentage of work/execution phase completed |
| CompletePercent | nvarchar | Percentage of completion/closeout phase completed |
| Total | nvarchar | Total count or measure of assignment items |
| Complete | nvarchar | Count or measure of completed assignment items |
| LastSyncTimestamp | nvarchar | Timestamp when data was last synchronized from source system |
| SnapshotTimestamp | nvarchar | Date/time when this progress snapshot was captured |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| LastLoaded | datetime2 | Timestamp of last modification, used for incremental loading |

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
    [CommandName] nvarchar(MAX) NULL,
    [Count] nvarchar(MAX) NULL,
    [SnapshotTimestamp] nvarchar(MAX) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [LastLoaded] datetime2(7) NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| UserId | uniqueidentifier | User who executed the commands |
| CommandName | nvarchar | Name or identifier of the command/action executed |
| Count | nvarchar | Number of times the command was executed in the snapshot period |
| SnapshotTimestamp | nvarchar | Hourly timestamp marking when this aggregation was captured |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| LastLoaded | datetime2 | Timestamp of last modification, used for incremental loading |

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
    [CommandName] nvarchar(MAX) NULL,
    [AssignmentId] uniqueidentifier NULL,
    [FiredTimestamp] nvarchar(MAX) NULL,
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
| CommandName | nvarchar | Name of the command or action that was audited |
| AssignmentId | uniqueidentifier | Assignment that was affected by the action |
| FiredTimestamp | nvarchar | Exact date/time when the audited event occurred |
| LastUpdated | datetime2 | Timestamp of last modification to this audit record |
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
    [LastOperatedAt] datetime2(7) NULL,
    [TimeSeriesTypeInstanceId] uniqueidentifier NULL,
    [SeriesInstanceName] nvarchar(MAX) NULL,
    [SeriesName] nvarchar(MAX) NULL,
    [SeriesIdentifier] nvarchar(MAX) NULL,
    [SequenceNumber] int NULL,
    [GroupFragmentReference] nvarchar(MAX) NULL,
    [CompletedAt] datetime2(7) NULL,
    [CompletedBy] nvarchar(MAX) NULL,
    [ParentId] uniqueidentifier NULL,
    [CreatedDate] datetime2(7) NULL,
    [LastUpdated] datetime2(7) NULL,
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
| LastOperatedAt | datetime2 | Date/time timestamp |
| TimeSeriesTypeInstanceId | uniqueidentifier | Links to the TimeSeriesTypeInstance record |
| SeriesInstanceName | nvarchar | Data field |
| SeriesName | nvarchar | Data field |
| SeriesIdentifier | nvarchar | Data field |
| SequenceNumber | int | Data field |
| GroupFragmentReference | nvarchar | Data field |
| CompletedAt | datetime2 | Date/time timestamp |
| CompletedBy | nvarchar | Data field |
| ParentId | uniqueidentifier | Links to the Parent record |
| CreatedDate | datetime2 | Date/time when this record was created |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
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
    [CapturedOn] datetime2(7) NULL,
    [CompletedAt] datetime2(7) NULL,
    [CapturedBy] nvarchar(MAX) NULL,
    [CompletedBy] nvarchar(MAX) NULL,
    [FieldMeasurementName] nvarchar(MAX) NULL,
    [New_Name] nvarchar(MAX) NULL,
    [FieldMeasurementName] nvarchar(MAX) NULL,
    [FieldMeasurementIdentifier] nvarchar(MAX) NULL,
    [SectionName] nvarchar(MAX) NULL,
    [SectionIdentifier] nvarchar(MAX) NULL,
    [DataType] nvarchar(MAX) NULL,
    [AssignmentId] uniqueidentifier NULL,
    [LowerBoundary] nvarchar(MAX) NULL,
    [UpperBoundary] nvarchar(MAX) NULL,
    [Preface] nvarchar(MAX) NULL,
    [Postface] nvarchar(MAX) NULL,
    [Unit] nvarchar(MAX) NULL,
    [Reading] decimal(18,2) NULL,
    [SelectedMultiSelectValue] nvarchar(MAX) NULL,
    [CreatedDate] datetime2(7) NULL,
    [LastUpdated] datetime2(7) NULL,
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
| Comments | nvarchar | Additional notes or comments |
| CapturedOn | datetime2 | Date/time timestamp |
| CompletedAt | datetime2 | Date/time timestamp |
| CapturedBy | nvarchar | Data field |
| CompletedBy | nvarchar | Data field |
| FieldMeasurementName | nvarchar | Data field |
| New_Name | nvarchar | Data field |
| FieldMeasurementName | nvarchar | Data field |
| FieldMeasurementIdentifier | nvarchar | Data field |
| SectionName | nvarchar | Data field |
| SectionIdentifier | nvarchar | Data field |
| DataType | nvarchar | Data field |
| AssignmentId | uniqueidentifier | Links to the Assignment record |
| LowerBoundary | nvarchar | Data field |
| UpperBoundary | nvarchar | Data field |
| Preface | nvarchar | Data field |
| Postface | nvarchar | Data field |
| Unit | nvarchar | Data field |
| Reading | decimal | Data field |
| SelectedMultiSelectValue | nvarchar | Data field |
| CreatedDate | datetime2 | Date/time when this record was created |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
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
    [AssignmentCode] nvarchar(MAX) NULL,
    [AssignmentPointId] uniqueidentifier NULL,
    [AssignedTo] nvarchar(MAX) NULL,
    [FromDate] datetime2(7) NULL,
    [ToDate] datetime2(7) NULL,
    [Status] nvarchar(MAX) NULL,
    [CreatedBy] nvarchar(MAX) NULL,
    [WorkTemplateId] uniqueidentifier NULL,
    [TeamId] uniqueidentifier NULL,
    [CompletedBy] nvarchar(MAX) NULL,
    [FinalisedBy] nvarchar(MAX) NULL,
    [CompletedOn] datetime2(7) NULL,
    [FinalisedOn] datetime2(7) NULL,
    [CancelledOn] datetime2(7) NULL,
    [DeclinedOn] datetime2(7) NULL,
    [RequiredDate] datetime2(7) NULL,
    [AssignmentCategoryId] uniqueidentifier NULL,
    [AssignmentTitle] nvarchar(MAX) NULL,
    [Revision] int NULL,
    [Effort] decimal(18,2) NULL,
    [CreatedDate] datetime2(7) NULL,
    [LastUpdated] datetime2(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| Id | uniqueidentifier | Unique identifier for this record |
| AssignmentCode | nvarchar | Data field |
| AssignmentPointId | uniqueidentifier | Links to the AssignmentPoint record |
| AssignedTo | nvarchar | Data field |
| FromDate | datetime2 | Date/time timestamp |
| ToDate | datetime2 | Date/time timestamp |
| Status | nvarchar | Current status of the record |
| CreatedBy | nvarchar | User who created this record |
| WorkTemplateId | uniqueidentifier | Links to the WorkTemplate record |
| TeamId | uniqueidentifier | Links to the Team record |
| CompletedBy | nvarchar | Data field |
| FinalisedBy | nvarchar | Data field |
| CompletedOn | datetime2 | Date/time timestamp |
| FinalisedOn | datetime2 | Date/time timestamp |
| CancelledOn | datetime2 | Date/time timestamp |
| DeclinedOn | datetime2 | Date/time timestamp |
| RequiredDate | datetime2 | Date/time timestamp |
| AssignmentCategoryId | uniqueidentifier | Links to the AssignmentCategory record |
| AssignmentTitle | nvarchar | Data field |
| Revision | int | Data field |
| Effort | decimal | Data field |
| CreatedDate | datetime2 | Date/time when this record was created |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
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
    [DeclinedBy] nvarchar(MAX) NULL,
    [DeclinedOn] datetime2(7) NULL,
    [DeclinedByFullName] nvarchar(MAX) NULL,
    [ReasonForDecliningComment] nvarchar(MAX) NULL,
    [ReasonForDeclining] nvarchar(MAX) NULL,
    [CreatedDate] datetime2(7) NULL,
    [LastUpdated] datetime2(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| AssignmentId | uniqueidentifier | Links to the Assignment record |
| DeclinedBy | nvarchar | Data field |
| DeclinedOn | datetime2 | Date/time timestamp |
| DeclinedByFullName | nvarchar | Data field |
| ReasonForDecliningComment | nvarchar | Data field |
| ReasonForDeclining | nvarchar | Data field |
| CreatedDate | datetime2 | Date/time when this record was created |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
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
    [Priority] nvarchar(MAX) NULL,
    [RaisedAt] datetime2(7) NULL,
    [RaisedBy] nvarchar(MAX) NULL,
    [RaisedByName] nvarchar(MAX) NULL,
    [ResolvedAt] datetime2(7) NULL,
    [ResolvedBy] nvarchar(MAX) NULL,
    [ResolvedByName] nvarchar(MAX) NULL,
    [Comment] nvarchar(MAX) NULL,
    [CreatedDate] datetime2(7) NULL,
    [LastUpdated] datetime2(7) NULL,
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
| Priority | nvarchar | Data field |
| RaisedAt | datetime2 | Date/time timestamp |
| RaisedBy | nvarchar | Data field |
| RaisedByName | nvarchar | Data field |
| ResolvedAt | datetime2 | Date/time timestamp |
| ResolvedBy | nvarchar | Data field |
| ResolvedByName | nvarchar | Data field |
| Comment | nvarchar | Additional notes or comments |
| CreatedDate | datetime2 | Date/time when this record was created |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
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
    [CreatedDate] datetime2(7) NULL,
    [LastUpdated] datetime2(7) NULL,
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
| CreatedDate | datetime2 | Date/time when this record was created |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
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
    [AssignmentId] uniqueidentifier NULL,
    [Key] nvarchar(MAX) NULL,
    [Value] nvarchar(MAX) NULL,
    [IsVisible] bit NULL,
    [IsInteractive] bit NULL,
    [CreatedDate] datetime2(7) NULL,
    [LastUpdated] datetime2(7) NULL,
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
| AssignmentId | uniqueidentifier | Links to the Assignment record |
| Key | nvarchar | Data field |
| Value | nvarchar | Data field |
| IsVisible | bit | Indicates if visible condition is true |
| IsInteractive | bit | Indicates if interactive condition is true |
| CreatedDate | datetime2 | Date/time when this record was created |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
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
    [PointId] uniqueidentifier NULL,
    [PointName] nvarchar(MAX) NULL,
    [ParentId] uniqueidentifier NULL,
    [SubsiteId] uniqueidentifier NULL,
    [AssignmentPointTypeName] nvarchar(MAX) NULL,
    [APLatitude] nvarchar(MAX) NULL,
    [APLongitude] nvarchar(MAX) NULL,
    [CreatedDate] datetime2(7) NULL,
    [LastUpdated] datetime2(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| PointId | uniqueidentifier | Links to the Point record |
| PointName | nvarchar | Data field |
| ParentId | uniqueidentifier | Links to the Parent record |
| SubsiteId | uniqueidentifier | Links to the Subsite record |
| AssignmentPointTypeName | nvarchar | Data field |
| APLatitude | nvarchar | Data field |
| APLongitude | nvarchar | Data field |
| CreatedDate | datetime2 | Date/time when this record was created |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
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
    [AttributeName] nvarchar(MAX) NULL,
    [AttributeValue] nvarchar(MAX) NULL,
    [AttributeGroupName] nvarchar(MAX) NULL,
    [AssignmentPointId] uniqueidentifier NULL,
    [CreatedDate] datetime2(7) NULL,
    [LastUpdated] datetime2(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| AttributeName | nvarchar | Data field |
| AttributeValue | nvarchar | Data field |
| AttributeGroupName | nvarchar | Data field |
| AssignmentPointId | uniqueidentifier | Links to the AssignmentPoint record |
| CreatedDate | datetime2 | Date/time when this record was created |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
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
    [TimeSeriesFieldMeasurementID] nvarchar(MAX) NULL,
    [AssignmentTimeSeriesFieldMeasurementId] uniqueidentifier NULL,
    [ParentId] uniqueidentifier NULL,
    [FileName] nvarchar(MAX) NULL,
    [DocumentURL] nvarchar(MAX) NULL,
    [CreatedDate] datetime2(7) NULL,
    [LastUpdated] datetime2(7) NULL,
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| DocumentId | uniqueidentifier | Links to the Document record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| TimeSeriesFieldMeasurementID | nvarchar | Data field |
| AssignmentTimeSeriesFieldMeasurementId | uniqueidentifier | Links to the AssignmentTimeSeriesFieldMeasurement record |
| ParentId | uniqueidentifier | Links to the Parent record |
| FileName | nvarchar | Data field |
| DocumentURL | nvarchar | Data field |
| CreatedDate | datetime2 | Date/time when this record was created |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
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
    [Category] nvarchar(MAX) NULL,
    [Code] nvarchar(MAX) NULL,
    [Color] nvarchar(MAX) NULL,
    [Critical] nvarchar(MAX) NULL,
    [CreatedDate] datetime2(7) NULL,
    [LastUpdated] datetime2(7) NULL,
    [IsDeleted] bit NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| Category | nvarchar | Data field |
| Code | nvarchar | Unique code or identifier |
| Color | nvarchar | Data field |
| Critical | nvarchar | Data field |
| CreatedDate | datetime2 | Date/time when this record was created |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
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
    [Id] uniqueidentifier NULL,
    [Name] nvarchar(MAX) NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| Name | nvarchar | Display name |

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
    [Column1Name] nvarchar(MAX) NULL,
    [Column2Name] nvarchar(MAX) NULL,
    [CreatedDate] datetime2(7) NULL,
    [LastUpdated] datetime2(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| Column1Name | nvarchar | Data field |
| Column2Name | nvarchar | Data field |
| CreatedDate | datetime2 | Date/time when this record was created |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
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
    [SiteId] uniqueidentifier NULL,
    [SiteName] nvarchar(MAX) NULL,
    [SiteAddressLine1] nvarchar(MAX) NULL,
    [SiteAddressLine2] nvarchar(MAX) NULL,
    [SiteAddressLine3] nvarchar(MAX) NULL,
    [SiteAddressCity] nvarchar(MAX) NULL,
    [SiteAddressPostCode] nvarchar(MAX) NULL,
    [SiteLocationDescription] nvarchar(MAX) NULL,
    [SiteUsageDescription] nvarchar(MAX) NULL,
    [APLatitude] nvarchar(MAX) NULL,
    [APLongitude] nvarchar(MAX) NULL,
    [CreatedDate] datetime2(7) NULL,
    [LastUpdated] datetime2(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| SiteId | uniqueidentifier | Links to the Site record |
| SiteName | nvarchar | Data field |
| SiteAddressLine1 | nvarchar | Data field |
| SiteAddressLine2 | nvarchar | Data field |
| SiteAddressLine3 | nvarchar | Data field |
| SiteAddressCity | nvarchar | Data field |
| SiteAddressPostCode | nvarchar | Data field |
| SiteLocationDescription | nvarchar | Data field |
| SiteUsageDescription | nvarchar | Data field |
| APLatitude | nvarchar | Data field |
| APLongitude | nvarchar | Data field |
| CreatedDate | datetime2 | Date/time when this record was created |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
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
    [SubSiteId] uniqueidentifier NULL,
    [SubSiteName] nvarchar(MAX) NULL,
    [SubSiteAddressLine1] nvarchar(MAX) NULL,
    [SubSiteAddressLine2] nvarchar(MAX) NULL,
    [SubSiteAddressLine3] nvarchar(MAX) NULL,
    [SubSiteAddressCity] nvarchar(MAX) NULL,
    [SubSiteAddressPostCode] nvarchar(MAX) NULL,
    [SubSiteLocationDescription] nvarchar(MAX) NULL,
    [SubSiteUsageDescription] nvarchar(MAX) NULL,
    [APLatitude] nvarchar(MAX) NULL,
    [APLongitude] nvarchar(MAX) NULL,
    [ParentSiteId] uniqueidentifier NULL,
    [CreatedDate] datetime2(7) NULL,
    [LastUpdated] datetime2(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| SubSiteId | uniqueidentifier | Links to the SubSite record |
| SubSiteName | nvarchar | Data field |
| SubSiteAddressLine1 | nvarchar | Data field |
| SubSiteAddressLine2 | nvarchar | Data field |
| SubSiteAddressLine3 | nvarchar | Data field |
| SubSiteAddressCity | nvarchar | Data field |
| SubSiteAddressPostCode | nvarchar | Data field |
| SubSiteLocationDescription | nvarchar | Data field |
| SubSiteUsageDescription | nvarchar | Data field |
| APLatitude | nvarchar | Data field |
| APLongitude | nvarchar | Data field |
| ParentSiteId | uniqueidentifier | Links to the ParentSite record |
| CreatedDate | datetime2 | Date/time when this record was created |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
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
    [Name] nvarchar(MAX) NULL,
    [Description] nvarchar(MAX) NULL,
    [CreatedDate] datetime2(7) NULL,
    [LastUpdated] datetime2(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [IsDeleted] bit NULL,
    [Name] nvarchar(MAX) NULL,
    [Name] nvarchar(MAX) NULL,
    [ResourceCentre] nvarchar(MAX) NULL,
    [Name] nvarchar(MAX) NULL,
    [Name] nvarchar(MAX) NULL,
    [Name] nvarchar(MAX) NULL,
    [WorkCentre] nvarchar(MAX) NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| Name | nvarchar | Display name |
| Description | nvarchar | Detailed description or notes |
| CreatedDate | datetime2 | Date/time when this record was created |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |
| Name | nvarchar | Display name |
| Name | nvarchar | Display name |
| ResourceCentre | nvarchar | Data field |
| Name | nvarchar | Display name |
| Name | nvarchar | Display name |
| Name | nvarchar | Display name |
| WorkCentre | nvarchar | Data field |

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
    [Name] nvarchar(MAX) NULL,
    [Identifier] nvarchar(MAX) NULL,
    [AccessControlled] nvarchar(MAX) NULL,
    [CreatedDate] datetime2(7) NULL,
    [LastUpdated] datetime2(7) NULL,
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
| Name | nvarchar | Display name |
| Identifier | nvarchar | Unique code or identifier |
| AccessControlled | nvarchar | Data field |
| CreatedDate | datetime2 | Date/time when this record was created |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
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
    [UserCode] nvarchar(MAX) NULL,
    [Email] nvarchar(MAX) NULL,
    [FullName] nvarchar(MAX) NULL,
    [IsActive] bit NULL,
    [LastUpdated] datetime2(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [ReferenceCode] nvarchar(MAX) NULL,
    [AuthorisationCode] nvarchar(MAX) NULL,
    [Department] nvarchar(MAX) NULL,
    [DepartmentCode] nvarchar(MAX) NULL,
    [Organisation] nvarchar(MAX) NULL,
    [OrganisationCode] nvarchar(MAX) NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| Id | uniqueidentifier | Unique identifier for this record |
| UserId | uniqueidentifier | Links to the User record |
| Role | nvarchar | Data field |
| UserCode | nvarchar | Data field |
| Email | nvarchar | Email address |
| FullName | nvarchar | Complete display name |
| IsActive | bit | Indicates if active condition is true |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| ReferenceCode | nvarchar | Data field |
| AuthorisationCode | nvarchar | Data field |
| Department | nvarchar | Data field |
| DepartmentCode | nvarchar | Data field |
| Organisation | nvarchar | Data field |
| OrganisationCode | nvarchar | Data field |

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
    [Identifier] nvarchar(MAX) NULL,
    [Name] nvarchar(MAX) NULL,
    [Version] nvarchar(MAX) NULL,
    [TemplateLink] nvarchar(MAX) NULL,
    [FragmentType] nvarchar(MAX) NULL,
    [CreatedDate] datetime2(7) NULL,
    [LastUpdated] datetime2(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [IsPublished] bit NULL,
    [Json] nvarchar(MAX) NULL,
    [bit] nvarchar(MAX) NULL,
    [nvarchar] nvarchar(MAX) NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| Identifier | nvarchar | Unique code or identifier |
| Name | nvarchar | Display name |
| Version | nvarchar | Data field |
| TemplateLink | nvarchar | Data field |
| FragmentType | nvarchar | Data field |
| CreatedDate | datetime2 | Date/time when this record was created |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| IsPublished | bit | Indicates if published condition is true |
| Json | nvarchar | Data field |
| bit | nvarchar | Data field |
| nvarchar | nvarchar | Data field |

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
    [Column1] nvarchar(MAX) NULL,
    [Column2] nvarchar(MAX) NULL,
    [Reference] nvarchar(MAX) NULL,
    [CreatedDate] datetime2(7) NULL,
    [LastUpdated] datetime2(7) NULL,
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
| Column1 | nvarchar | Data field |
| Column2 | nvarchar | Data field |
| Reference | nvarchar | Data field |
| CreatedDate | datetime2 | Date/time when this record was created |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
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
    [CreatedDate] datetime2(7) NULL,
    [LastUpdated] datetime2(7) NULL,
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
| CreatedDate | datetime2 | Date/time when this record was created |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
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
    [TemplateLink] nvarchar(MAX) NULL,
    [LastUpdated] datetime2(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [CreatedDate] datetime2(7) NULL,
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for this record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| TemplateGroupId | uniqueidentifier | Links to the TemplateGroup record |
| TemplateLink | nvarchar | Data field |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| CreatedDate | datetime2 | Date/time when this record was created |
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
    [FileName] nvarchar(MAX) NULL,
    [DocumentKind] nvarchar(MAX) NULL,
    [TenantId] uniqueidentifier NULL,
    [AssignmentTimeSeriesFieldMeasurementSectionId] uniqueidentifier NULL,
    [TimeSeriesId] uniqueidentifier NULL,
    [DocumentURL] nvarchar(MAX) NULL,
    [CreatedDate] datetime2(7) NULL,
    [LastUpdated] datetime2(7) NULL,
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| DocumentId | uniqueidentifier | Links to the Document record |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| ParentId | uniqueidentifier | Links to the Parent record |
| FileName | nvarchar | Data field |
| DocumentKind | nvarchar | Data field |
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| AssignmentTimeSeriesFieldMeasurementSectionId | uniqueidentifier | Links to the AssignmentTimeSeriesFieldMeasurementSection record |
| TimeSeriesId | uniqueidentifier | Links to the TimeSeries record |
| DocumentURL | nvarchar | Data field |
| CreatedDate | datetime2 | Date/time when this record was created |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
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
    [LabelKey] nvarchar(MAX) NULL,
    [LabelValue] nvarchar(MAX) NULL,
    [LabelPluralValue] nvarchar(MAX) NULL,
    [CreatedDate] datetime2(7) NULL,
    [LastUpdated] datetime2(7) NULL,
    [PagingId] binary(8) NULL,  -- Converted from rowversion
    [IsDeleted] bit NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| Id | uniqueidentifier | Unique identifier for this record |
| LabelKey | nvarchar | Data field |
| LabelValue | nvarchar | Data field |
| LabelPluralValue | nvarchar | Data field |
| CreatedDate | datetime2 | Date/time when this record was created |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
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
    [Timezone] nvarchar(MAX) NULL,
    [LogoBase64] nvarchar(MAX) NULL,
    [TenantName] nvarchar(MAX) NULL,
    [TenantURL] nvarchar(MAX) NULL,
    [PrimaryColour] nvarchar(MAX) NULL,
    [SecondaryColour] nvarchar(MAX) NULL,
    [AssignmentPointList1] nvarchar(MAX) NULL,
    [AssignmentPointList2] nvarchar(MAX) NULL,
    [TempleteList1] nvarchar(MAX) NULL,
    [TempleteList2] nvarchar(MAX) NULL,
    [SeriesList1] nvarchar(MAX) NULL,
    [SeriesList2] nvarchar(MAX) NULL,
    [OrganisationKey] nvarchar(MAX) NULL
);
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| TenantId | uniqueidentifier | Unique identifier for the tenant organization |
| Timezone | nvarchar | Data field |
| LogoBase64 | nvarchar | Data field |
| TenantName | nvarchar | Data field |
| TenantURL | nvarchar | Data field |
| PrimaryColour | nvarchar | Data field |
| SecondaryColour | nvarchar | Data field |
| AssignmentPointList1 | nvarchar | Data field |
| AssignmentPointList2 | nvarchar | Data field |
| TempleteList1 | nvarchar | Data field |
| TempleteList2 | nvarchar | Data field |
| SeriesList1 | nvarchar | Data field |
| SeriesList2 | nvarchar | Data field |
| OrganisationKey | nvarchar | Data field |

---

