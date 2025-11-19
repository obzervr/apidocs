# TimeSeries

**Table Type:** Fact Table (Incremental Refresh)  
**Schema:** dbo.VW_TimeSeries  
**Primary Key:** Id  
**Related ERD:** [ERD #3: Field Measurements & Time Series](../ERDs/ERD_03_Field_Measurements_Time_Series.md)

---

## Table Overview

Contains time series instances for assignments, representing repeatable data collection groups (e.g., equipment readings, inspection rounds, work sequences). Each row represents a specific instance of a time series type with parent-child relationships for hierarchical organization. Supports incremental refresh based on Created_Date.

**Source System:** Analytic Database (dbo.VW_TimeSeries view)

**Row Count:** Variable (grows with assignment activity; typically 100,000-1,000,000+ rows)

**Refresh Type:** Incremental refresh (5-year rolling window, 1-year incremental periods)

**Multi-Tenant:** Yes (filtered by Tenant_Id)

**Incremental Refresh:** Created_Date >= RangeStart and < RangeEnd

---

## Table Specifications

| Property | Value |
|----------|-------|
| Lineage Tag | 1ab42279-47d8-4923-a350-be77512ca400 |
| Query Group | Assignments |
| Partitions | 1 (incremental refresh) |
| Relationships | Outbound to Assignments, Dim_Tenant; inbound from TimeSeries_FieldMeasurements |
| Calculated Columns | 1 (URL) |
| Hierarchies | 0 (parent-child via ParentId) |

---

## Column Specifications

| Column Name | Data Type | Format | Description | Nullable | Hidden |
|-------------|-----------|--------|-------------|----------|--------|
| **Tenant_Id** | string | - | Tenant identifier | No | Yes |
| **Id** | string | - | Unique timeseries instance identifier | No | Yes |
| **Assignment_Id** | string | - | Parent assignment identifier | No | Yes |
| **Last_Updated** | datetime | General Date | Last update timestamp | Yes | Yes |
| **Last_Operated_At** | datetime | dd/MM/yyyy HH:mm | Last operation timestamp | Yes | Yes |
| **TimeSeries_Type_Instance_Id** | string | - | Type instance identifier | Yes | Yes |
| **Created_Date** | datetime | General Date | Creation timestamp (incremental refresh key) | Yes | Yes |
| **Series_Instance_Name** | string | - | Instance display name | Yes | Yes |
| **Series_Name** | string | - | Series type name | Yes | Yes |
| **Series_Identifier** | string | - | Business identifier for series | Yes | Yes |
| **Sequence_Number** | int64 | 0 | Ordering sequence | Yes | Yes |
| **Group_Fragment_Reference** | string | - | Fragment reference for grouping | Yes | Yes |
| **Completed_At** | datetime | dd/MM/yyyy HH:mm | Completion timestamp | Yes | Yes |
| **Completed_By** | string | - | User who completed series | Yes | Yes |
| **URL** | string | - | Deep link URL to edit timeseries (calculated) | No | Yes |
| **ParentId** | string | - | Parent timeseries ID (self-referencing for hierarchy) | Yes | No |

**Note:** All columns except URL and ParentId are hidden from report view. URL is calculated. ParentId enables parent-child hierarchy.

---

## Calculated Columns

### URL

**Purpose:** Generate deep link URL to edit timeseries in Obzervr application

**Data Type:** String (Web URL)

**Data Category:** WebUrl

**DAX Expression:**
```dax
URL = 
MAX(Dim_Tenant[Tenant_URL]) 
& "/#/assignments/edit/" 
& RELATED(Assignments[Assignment_Id]) 
& "/obzervtimeseriestype/" 
& TimeSeries[TimeSeries_Type_Instance_Id] 
& "/modal"
```

**Logic:**
1. Get tenant base URL from Dim_Tenant
2. Construct deep link with assignment ID and timeseries type instance ID
3. Append "/modal" for modal dialog display

**Use Cases:**
- Direct navigation from Power BI reports to specific timeseries edit screens
- Audit trail hyperlinks
- User action buttons in dashboards

**Example URL:** `https://app.obzervr.com/#/assignments/edit/ABC123/obzervtimeseriestype/XYZ789/modal`

---

## Relationships

### Outbound Relationships

**To Dim_Tenant**
- **Type:** Many-to-one
- **From Column:** Tenant_Id
- **To Column:** Dim_Tenant[Tenant_Id]
- **Purpose:** Links timeseries to tenant configuration

**To Assignments**
- **Type:** Many-to-one
- **From Column:** Assignment_Id
- **To Column:** Assignments[Assignment_Id]
- **Purpose:** Links timeseries instances to parent assignments

**To TimeSeries (Self-Referencing)**
- **Type:** Parent-Child
- **From Column:** ParentId
- **To Column:** TimeSeries[Id]
- **Purpose:** Enables hierarchical timeseries groups (e.g., Fleet → Truck Type → Individual Truck)

### Inbound Relationships

**From TimeSeries_FieldMeasurements**
- **Type:** One-to-many
- **From Column:** TimeSeries_FieldMeasurements[TimeSeries_Id]
- **To Column:** TimeSeries[Id]
- **Purpose:** Links field measurement readings to timeseries instances

---

## Power Query M Source

```m
let
    Source = Sql.Database(#"SQL Server", #"Analytic Database", [CreateNavigationProperties=false, MultiSubnetFailover=true]),
    Table = Source{[Schema="dbo",Item="VW_TimeSeries"]}[Data],
    #"Filtered Rows" = Table.SelectRows(Table, each AllTenants = true or [TenantId] = #"TenantId1" or [TenantId] = TenantId2 or [TenantId] = TenantId3 or [TenantId] = TenantId4 or [TenantId] = TenantId5),
    #"Renamed Columns" = Table.RenameColumns(#"Filtered Rows",{{"TenantId", "Tenant_Id"}, {"AssignmentId", "Assignment_Id"}, {"CreatedDate", "Created_Date"}, {"LastUpdated", "Last_Updated"}, {"LastOperatedAt", "Last_Operated_At"}, {"TimeSeriesTypeInstanceId", "TimeSeries_Type_Instance_Id"}, {"SeriesInstanceName", "Series_Instance_Name"}, {"SeriesName", "Series_Name"}, {"SeriesIdentifier", "Series_Identifier"}, {"SequenceNumber", "Sequence_Number"}, {"GroupFragmentReference", "Group_Fragment_Reference"}, {"CompletedAt", "Completed_At"}, {"CompletedBy", "Completed_By"}}),
    Incremental.Refresh = Table.SelectRows(#"Renamed Columns", each [Created_Date] >= RangeStart and [Created_Date] < RangeEnd)
in
    Incremental.Refresh
```

**Incremental Refresh Policy:**
- **Rolling Window:** 5 years
- **Incremental Period:** 1 year
- **Filter Column:** Created_Date
- **Range Variables:** RangeStart and RangeEnd set by Power BI service

---

## DAX Query Patterns

### Timeseries hierarchy (parent-child)

```dax
EVALUATE
ADDCOLUMNS(
    FILTER(TimeSeries, ISBLANK([ParentId])),
    "Series", [Series_Name],
    "Instance", [Series_Instance_Name],
    "Child Count", COUNTROWS(RELATEDTABLE(FILTER(TimeSeries, TimeSeries[ParentId] = EARLIER(TimeSeries[Id]))))
)
ORDER BY [Series_Name]
```

### Timeseries by assignment

```dax
EVALUATE
SUMMARIZECOLUMNS(
    Assignments[Assignment_Code],
    TimeSeries[Series_Name],
    "Series Count", COUNT(TimeSeries[Id]),
    "Completed Count", CALCULATE(COUNT(TimeSeries[Id]), NOT(ISBLANK(TimeSeries[Completed_At])))
)
ORDER BY [Assignment_Code], [Series_Name]
```

### Active vs completed timeseries

```dax
EVALUATE
SUMMARIZECOLUMNS(
    "Total TimeSeries", COUNT(TimeSeries[Id]),
    "Completed", CALCULATE(COUNT(TimeSeries[Id]), NOT(ISBLANK(TimeSeries[Completed_At]))),
    "In Progress", CALCULATE(COUNT(TimeSeries[Id]), ISBLANK(TimeSeries[Completed_At])),
    "Completion %", DIVIDE(
        CALCULATE(COUNT(TimeSeries[Id]), NOT(ISBLANK(TimeSeries[Completed_At]))),
        COUNT(TimeSeries[Id]),
        0
    )
)
```

---

## Data Model Pattern

**Pattern:** Incremental Refresh Fact Table with Parent-Child Hierarchy

**Characteristics:**
- **Parent-Child Hierarchy**: ParentId self-references Id for hierarchical groups
- **Assignment-Scoped**: Each timeseries belongs to one assignment
- **Incremental Refresh**: 5-year rolling window with 1-year increments based on Created_Date
- **Hidden Columns**: Most columns hidden from report view (analysis via relationships and measures)
- **Deep Linking**: URL column enables navigation to Obzervr application

**Parent-Child Hierarchy Example:**
```
Mining Fleet (ParentId = NULL, Series_Name = "Mining Fleet")
  ├─ Haul Trucks (ParentId = Mining Fleet Id, Series_Name = "Haul Truck Type")
  │   ├─ Truck 001 (ParentId = Haul Trucks Id, Series_Instance_Name = "Truck 001")
  │   ├─ Truck 002 (ParentId = Haul Trucks Id, Series_Instance_Name = "Truck 002")
  │   └─ Truck 003
  └─ Excavators (ParentId = Mining Fleet Id)
      ├─ Excavator A
      └─ Excavator B
```

**TimeSeries vs TimeSeries_FieldMeasurements:**
- **TimeSeries**: Container/group for related measurements (the "what" being measured)
- **TimeSeries_FieldMeasurements**: Individual reading values (the actual measurements)
- **Relationship**: One timeseries → Many field measurements

**Use Cases:**
- Equipment inspection rounds (parent: equipment type, children: individual equipment)
- Shift readings (parent: shift, children: hourly readings)
- Multi-stage processes (parent: process, children: stages)
- Hierarchical checklists (parent: main checklist, children: sub-checklists)

---

## Related Documentation

### ERD Documents
- [ERD #3: Field Measurements & Time Series](../ERDs/ERD_03_Field_Measurements_Time_Series.md) - Parent-child hierarchy pattern

### Related Tables
- **TimeSeries_FieldMeasurements** - Child table with actual measurement readings
- **Assignments** - Parent assignment for each timeseries
- **Dim_Tenant** - Tenant configuration for URL generation

### Other Documentation
- [ERD_Overview.md](../ERD_Overview.md) - Incremental refresh patterns

---

## Change History

| Date | Change Description | Modified By |
|------|-------------------|-------------|
| 2025-11-18 | Initial table documentation created from TMDL metadata | AI Documentation Generator |

---

## Notes

**Incremental Refresh:** 5-year rolling window keeps recent data in memory while archiving older data to reduce model size and improve refresh performance.

**Hidden Columns:** Most columns hidden because analysis typically occurs through relationships to TimeSeries_FieldMeasurements and Assignments rather than direct timeseries table analysis.

**Parent-Child Hierarchy:** Use ParentId to build hierarchical visualizations. Top-level timeseries have ParentId = NULL. Use DAX PATH functions for multi-level hierarchies.

**URL Column:** Requires Dim_Tenant[Tenant_URL] to be populated correctly. URL format matches Obzervr application routing structure for deep linking.

**Sequence_Number:** Used for ordering timeseries instances within groups (e.g., Step 1, Step 2, Step 3 in sequential processes).

**Completed_At vs Last_Operated_At:** Completed_At indicates final completion; Last_Operated_At tracks most recent activity (may be updated multiple times before completion).

**Group_Fragment_Reference:** Links timeseries to template fragment definitions in Dim_Fragments table, enabling template-driven reporting.
