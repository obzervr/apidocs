# Obzervr Data Pipeline - Data Reference Catalog

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
| [FactAssignmentDetailsSnapshot](#factassignmentdetailssnapshot) | Extra Large | `pipeline.sp_GeAssignmentDetailsSnapshot` | 100,000 | Yes | No |
| [FactAssignmentProgressSnapshot](#factassignmentprogresssnapshot) | Extra Large | `pipeline.sp_GetAssignmentProgressSnapshot` | 100,000 | Yes | No |
| [FactAuditCommandCountSnapshotHourly](#factauditcommandcountsnapshothourly) | Extra Large | `pipeline.sp_GetFactAuditCommandCountSnapshotHourly` | 100,000 | Yes | No |
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
| [FactTemplateGroupWorkTemplates](#facttemplategroupworktemplates) | Small | `pipeline.sp_GetTemplateGroupWorkTemplates` | 100 | Yes | No |
| [FactTimeSeriesSectionDocuments](#facttimeseriessectiondocuments) | Small | `pipeline.sp_GetTimeSeriesSectionDocuments` | 10,000 | Yes | No |
| [LabelAlias](#labelalias) | Small | `pipeline.sp_GetLabelAlias` | 100 | No | Yes |
| [TenantSettings](#tenantsettings) | Small | `pipeline.sp_GetTenants` | 10 | No | No |

---

### FactAssignmentDetailsSnapshot

**Stored Procedure:** `pipeline.sp_GeAssignmentDetailsSnapshot`

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

#### Usage Example

```sql
exec pipeline.sp_GeAssignmentDetailsSnapshot
@LastPagingId = 0x0000000000003030,
@LastUpdatedStart = '2025-10-01’
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| AssignmentId | uniqueidentifier | Foreign key reference to related Assignment record |
| UserId | uniqueidentifier | Foreign key reference to related User record |
| TeamId | uniqueidentifier | Foreign key reference to related Team record |
| Status | nvarchar | Current status of the record |
| AssignedTo | nvarchar | Data field |
| FromDate | datetime2 | Date/time timestamp |
| ToDate | datetime2 | Date/time timestamp |
| RequiredDate | datetime2 | Date/time timestamp |
| SnapshotTimestamp | nvarchar | Data field |
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

**Notes:** This snapshot is only for work order complete% purpose, I cannot see any use case for analytics

#### Usage Example

```sql
exec pipeline.sp_GetAssignmentProgressSnapshot
@LastPagingId = 0x00000000002C7DF7,
@LastUpdatedStart = '2025-10-01’
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| AssignmentId | uniqueidentifier | Foreign key reference to related Assignment record |
| TotalPercent | nvarchar | Data field |
| PlanPercent | nvarchar | Data field |
| WorkPercent | nvarchar | Data field |
| CompletePercent | nvarchar | Data field |
| Total | nvarchar | Data field |
| Complete | nvarchar | Data field |
| LastSyncTimestamp | nvarchar | Data field |
| SnapshotTimestamp | nvarchar | Data field |
| PagingId | rowversion | Rowversion used for paging through large result sets |
| LastLoaded | datetime2 | Timestamp of last modification, used for incremental loading |

---


## Stored Procedure Details


### FactAuditCommandCountSnapshotHourly

**Stored Procedure:** `pipeline.sp_GetFactAuditCommandCountSnapshotHourly`

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

#### Usage Example

```sql
exec pipeline.sp_GetAuditCommandCountSnapshotHourly
@LastPagingId = 0x00000000002EE0CB,
@LastUpdatedStart = '2025-10-01’
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| UserId | uniqueidentifier | Foreign key reference to related User record |
| CommandName | nvarchar | Data field |
| Count | nvarchar | Data field |
| SnapshotTimestamp | nvarchar | Data field |
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

#### Usage Example

```sql
exec [pipeline].sp_GetAuditsUserAssignment
@LastPagingId = 0x00000000001FA570,
@LastUpdatedStart = '2025-10-05’
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| Id | uniqueidentifier | Unique identifier for the record |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| UserId | uniqueidentifier | Foreign key reference to related User record |
| CommandName | nvarchar | Data field |
| AssignmentId | uniqueidentifier | Foreign key reference to related Assignment record |
| FiredTimestamp | nvarchar | Data field |

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

#### Usage Example

```sql
exec pipeline.sp_GetTimeSeries
@LastPagingId =0x0000000000003030,
@LastUpdatedStart = '2025-10-05’
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| Id | uniqueidentifier | Unique identifier for the record |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| AssignmentId | uniqueidentifier | Foreign key reference to related Assignment record |
| LastOperatedAt | datetime2 | Date/time timestamp |
| TimeSeriesTypeInstanceId | uniqueidentifier | Foreign key reference to related TimeSeriesTypeInstance record |
| SeriesInstanceName | nvarchar | Data field |
| SeriesName | nvarchar | Data field |
| SeriesIdentifier | nvarchar | Data field |
| SequenceNumber | int | Data field |
| GroupFragmentReference | nvarchar | Data field |
| CompletedAt | datetime2 | Date/time timestamp |
| CompletedBy | nvarchar | Data field |
| ParentId | uniqueidentifier | Foreign key reference to related Parent record |
| CreatedDate | datetime2 | Timestamp when the record was originally created |
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

#### Usage Example

```sql
exec pipeline.sp_GetTimeSeriesFieldMeasurements
@LastPagingId = 0x0000000000003030,
@LastUpdatedStart = '2025-10-05’
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| FieldMeasurementName | nvarchar | Data field |
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| Id | uniqueidentifier | Unique identifier for the record |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| TimeSeriesId | uniqueidentifier | Foreign key reference to related TimeSeries record |
| Comments | nvarchar | Data field |
| CapturedOn | datetime2 | Date/time timestamp |
| CompletedAt | datetime2 | Date/time timestamp |
| CapturedBy | nvarchar | Data field |
| CompletedBy | nvarchar | Data field |
| FieldMeasurementIdentifier | nvarchar | Data field |
| SectionName | nvarchar | Data field |
| SectionIdentifier | nvarchar | Data field |
| DataType | nvarchar | Data field |
| AssignmentId | uniqueidentifier | Foreign key reference to related Assignment record |
| LowerBoundary | nvarchar | Data field |
| UpperBoundary | nvarchar | Data field |
| Reading | decimal | Data field |
| SelectedMultiSelectValue | nvarchar | Data field |
| CreatedDate | datetime2 | Timestamp when the record was originally created |
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

#### Usage Example

```sql
exec pipeline.sp_GetAssignments
@LastPagingId = 0x0000000000003030,
@LastUpdatedStart = '2025-10-05’
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| Id | uniqueidentifier | Unique identifier for the record |
| AssignmentCode | nvarchar | Data field |
| AssignmentPointId | uniqueidentifier | Foreign key reference to related AssignmentPoint record |
| AssignedTo | nvarchar | Data field |
| FromDate | datetime2 | Date/time timestamp |
| ToDate | datetime2 | Date/time timestamp |
| Status | nvarchar | Current status of the record |
| CreatedBy | nvarchar | User ID who created the record |
| WorkTemplateId | uniqueidentifier | Foreign key reference to related WorkTemplate record |
| TeamId | uniqueidentifier | Foreign key reference to related Team record |
| CompletedBy | nvarchar | Data field |
| FinalisedBy | nvarchar | Data field |
| CompletedOn | datetime2 | Date/time timestamp |
| FinalisedOn | datetime2 | Date/time timestamp |
| CancelledOn | datetime2 | Date/time timestamp |
| DeclinedOn | datetime2 | Date/time timestamp |
| RequiredDate | datetime2 | Date/time timestamp |
| AssignmentCategoryId | uniqueidentifier | Foreign key reference to related AssignmentCategory record |
| AssignmentTitle | nvarchar | Data field |
| Revision | int | Data field |
| Effort | decimal | Data field |
| CreatedDate | datetime2 | Timestamp when the record was originally created |
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

#### Usage Example

```sql
exec pipeline.sp_GetAssignmentDeclinedReasons
@LastPagingId = 0x0000000000003030,
@LastUpdatedStart = '2025-10-05’
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for the record |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| AssignmentId | uniqueidentifier | Foreign key reference to related Assignment record |
| DeclinedBy | nvarchar | Data field |
| DeclinedOn | datetime2 | Date/time timestamp |
| DeclinedByFullName | nvarchar | Data field |
| ReasonForDecliningComment | nvarchar | Data field |
| ReasonForDeclining | nvarchar | Data field |
| CreatedDate | datetime2 | Timestamp when the record was originally created |

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

#### Usage Example

```sql
exec pipeline.sp_GetAssignmentExceptions
@LastPagingId = 0x0000000000003030,
@LastUpdatedStart = '2025-10-05’
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |
| Id | uniqueidentifier | Unique identifier for the record |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| AssignmentId | uniqueidentifier | Foreign key reference to related Assignment record |
| FieldMeasureId | uniqueidentifier | Foreign key reference to related FieldMeasure record |
| Priority | nvarchar | Data field |
| RaisedAt | datetime2 | Date/time timestamp |
| RaisedBy | nvarchar | Data field |
| RaisedByName | nvarchar | Data field |
| ResolvedAt | datetime2 | Date/time timestamp |
| ResolvedBy | nvarchar | Data field |
| ResolvedByName | nvarchar | Data field |
| Comment | nvarchar | Data field |
| CreatedDate | datetime2 | Timestamp when the record was originally created |

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

#### Usage Example

```sql
exec pipeline.sp_GetAssignmentFieldOperators
@LastPagingId = 0x0000000000003030,
@LastUpdatedStart = '2025-10-05’
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| Id | uniqueidentifier | Unique identifier for the record |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| AssignmentId | uniqueidentifier | Foreign key reference to related Assignment record |
| FieldOperatorId | uniqueidentifier | Foreign key reference to related FieldOperator record |
| CreatedDate | datetime2 | Timestamp when the record was originally created |
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

#### Usage Example

```sql
exec pipeline.sp_GetAssignmentTags
@LastPagingId = 0x0000000000003030,
@LastUpdatedStart = '2025-10-05’
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| Id | uniqueidentifier | Unique identifier for the record |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| AssignmentId | uniqueidentifier | Foreign key reference to related Assignment record |
| AssignmentId | uniqueidentifier | Foreign key reference to related Assignment record |
| Key | nvarchar | Data field |
| Value | nvarchar | Data field |
| IsVisible | bit | Boolean flag |
| IsInteractive | bit | Boolean flag |
| CreatedDate | datetime2 | Timestamp when the record was originally created |
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

**Notes:** It is Large to Extra Large based on customers. Bluescope has Extra large data

#### Usage Example

```sql
exec pipeline.sp_GetAssignmentPoints
@LastPagingId = 0x0000000000003030,
@LastUpdatedStart = '2025-10-05’
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| Id | uniqueidentifier | Unique identifier for the record |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| PointId | uniqueidentifier | Foreign key reference to related Point record |
| PointName | nvarchar | Data field |
| ParentId | uniqueidentifier | Foreign key reference to related Parent record |
| SubsiteId | uniqueidentifier | Foreign key reference to related Subsite record |
| AssignmentPointTypeName | nvarchar | Data field |
| APLatitude | nvarchar | Data field |
| APLongitude | nvarchar | Data field |
| CreatedDate | datetime2 | Timestamp when the record was originally created |
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

**Notes:** It is Large to Extra Large based on customers. Bluescope has Extra large data

#### Usage Example

```sql
exec pipeline.sp_GetAssignmentPointAttributes
@LastPagingId = 0x0000000000003030,
@LastUpdatedStart = '2025-10-05’
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| Id | uniqueidentifier | Unique identifier for the record |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| AttributeName | nvarchar | Data field |
| AttributeValue | nvarchar | Data field |
| AttributeGroupName | nvarchar | Data field |
| AssignmentPointId | uniqueidentifier | Foreign key reference to related AssignmentPoint record |
| CreatedDate | datetime2 | Timestamp when the record was originally created |

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

#### Usage Example

```sql
exec pipeline.sp_GetTimeSeriesFieldMeasurementDocuments
@LastPagingId = 0x0000000000003030,
@LastUpdatedStart = '2025-10-05’
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |
| DocumentId | uniqueidentifier | Foreign key reference to related Document record |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| TimeSeriesFieldMeasurementID | nvarchar | Data field |
| AssignmentTimeSeriesFieldMeasurementId | uniqueidentifier | Foreign key reference to related AssignmentTimeSeriesFieldMeasurement record |
| ParentId | uniqueidentifier | Foreign key reference to related Parent record |
| FileName | nvarchar | Data field |
| DocumentURL | nvarchar | Data field |
| CreatedDate | datetime2 | Timestamp when the record was originally created |

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

#### Usage Example

```sql
exec pipeline.sp_GetAssignmentCategories
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| Id | uniqueidentifier | Unique identifier for the record |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| Category | nvarchar | Data field |
| Code | nvarchar | Unique code or identifier |
| Color | nvarchar | Data field |
| Critical | nvarchar | Data field |
| CreatedDate | datetime2 | Timestamp when the record was originally created |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |

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

**Notes:** Lookup table only

#### Usage Example

```sql
exec pipeline.sp_GetAssignmentStatus
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| Id | uniqueidentifier | Unique identifier for the record |
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

#### Usage Example

```sql
exec pipeline.sp_GetFieldMeasurementTableDefinitions
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| Id | uniqueidentifier | Unique identifier for the record |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| Column1Name | nvarchar | Data field |
| Column2Name | nvarchar | Data field |
| CreatedDate | datetime2 | Timestamp when the record was originally created |
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

#### Usage Example

```sql
exec pipeline.sp_GetSites
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| Id | uniqueidentifier | Unique identifier for the record |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| SiteId | uniqueidentifier | Foreign key reference to related Site record |
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
| CreatedDate | datetime2 | Timestamp when the record was originally created |
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

#### Usage Example

```sql
exec pipeline.sp_GetSubSites
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| Id | uniqueidentifier | Unique identifier for the record |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| SubSiteId | uniqueidentifier | Foreign key reference to related SubSite record |
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
| ParentSiteId | uniqueidentifier | Foreign key reference to related ParentSite record |
| CreatedDate | datetime2 | Timestamp when the record was originally created |
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

#### Usage Example

```sql
exec pipeline.sp_GetTeams
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| Id | uniqueidentifier | Unique identifier for the record |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| Name | nvarchar | Display name |
| Description | nvarchar | Data field |
| CreatedDate | datetime2 | Timestamp when the record was originally created |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |
| Name | nvarchar | Display name |
| Name | nvarchar | Display name |
| Name | nvarchar | Display name |
| Name | nvarchar | Display name |
| Name | nvarchar | Display name |

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

#### Usage Example

```sql
exec pipeline.sp_GetTemplateGroupTemplates
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| Id | uniqueidentifier | Unique identifier for the record |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| CreatedByUserId | uniqueidentifier | Foreign key reference to related CreatedByUser record |
| UpdatedByUserId | uniqueidentifier | Foreign key reference to related UpdatedByUser record |
| Name | nvarchar | Display name |
| Identifier | nvarchar | Unique code or identifier |
| AccessControlled | nvarchar | Data field |
| CreatedDate | datetime2 | Timestamp when the record was originally created |
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

#### Usage Example

```sql
exec pipeline.sp_GetUsers
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| Id | uniqueidentifier | Unique identifier for the record |
| UserId | uniqueidentifier | Foreign key reference to related User record |
| Role | nvarchar | Data field |
| UserCode | nvarchar | Data field |
| Email | nvarchar | Data field |
| FullName | nvarchar | Data field |
| IsActive | bit | Boolean flag |
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

#### Usage Example

```sql
exec pipeline.sp_GetWorkTemplates
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| Id | uniqueidentifier | Unique identifier for the record |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| Identifier | nvarchar | Unique code or identifier |
| Name | nvarchar | Display name |
| Version | nvarchar | Data field |
| TemplateLink | nvarchar | Data field |
| FragmentType | nvarchar | Data field |
| CreatedDate | datetime2 | Timestamp when the record was originally created |
| IsPublished | bit | Boolean flag |
| Json | nvarchar | Data field |

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

#### Usage Example

```sql
exec pipeline.sp_GetFieldMeasurementTableContents
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| Id | uniqueidentifier | Unique identifier for the record |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| FieldMeasurementTableDefinitionId | uniqueidentifier | Foreign key reference to related FieldMeasurementTableDefinition record |
| Column1 | nvarchar | Data field |
| Column2 | nvarchar | Data field |
| Reference | nvarchar | Data field |
| CreatedDate | datetime2 | Timestamp when the record was originally created |
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

#### Usage Example

```sql
exec pipeline.sp_GetTeamUsers
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| UserId | uniqueidentifier | Foreign key reference to related User record |
| TeamId | uniqueidentifier | Foreign key reference to related Team record |
| IsSupervisor | bit | Boolean flag |
| IsMember | bit | Boolean flag |
| CreatedDate | datetime2 | Timestamp when the record was originally created |

---


### FactTemplateGroupWorkTemplates

**Stored Procedure:** `pipeline.sp_GetTemplateGroupWorkTemplates`

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

#### Usage Example

```sql
exec pipeline.sp_GetTemplateGroups
```

#### Result Set Schema

*Result set schema extraction pending. Please refer to stored procedure definition.*

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

#### Usage Example

```sql
exec pipeline.sp_GetTimeSeriesSectionDocuments
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| IsDeleted | bit | Soft delete flag indicating if record is logically deleted |
| DocumentId | uniqueidentifier | Foreign key reference to related Document record |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| ParentId | uniqueidentifier | Foreign key reference to related Parent record |
| FileName | nvarchar | Data field |
| DocumentKind | nvarchar | Data field |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| AssignmentTimeSeriesFieldMeasurementSectionId | uniqueidentifier | Foreign key reference to related AssignmentTimeSeriesFieldMeasurementSection record |
| TimeSeriesId | uniqueidentifier | Foreign key reference to related TimeSeries record |
| DocumentURL | nvarchar | Data field |
| CreatedDate | datetime2 | Timestamp when the record was originally created |

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

**Notes:** Tenant configuration table only

#### Usage Example

```sql
exec pipeline.sp_GetLabelAlias
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| LastUpdated | datetime2 | Timestamp of last modification, used for incremental loading |
| TenantId | uniqueidentifier | Unique identifier for the tenant |
| Id | uniqueidentifier | Unique identifier for the record |
| LabelKey | nvarchar | Data field |
| LabelValue | nvarchar | Data field |
| LabelPluralValue | nvarchar | Data field |
| CreatedDate | datetime2 | Timestamp when the record was originally created |
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

#### Usage Example

```sql
exec pipeline.sp_GetTenants
```

#### Result Set Schema

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| TenantId | uniqueidentifier | Unique identifier for the tenant |
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

