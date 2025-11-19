# Dim_Assignment_Status

**Table Type:** Dimension Table (Lookup)  
**Schema:** dbo.DimAssignmentStatus  
**Primary Key:** Id  
**Related ERD:** [ERD #1: Assignment Core Model](../ERDs/ERD_01_Assignment_Core.md)

---

## Table Overview

Small lookup dimension containing assignment status values with associated colors for visualization. Defines the assignment lifecycle stages (Draft, Active, Completed, Finalised, Archived, Cancelled) with color-coding for consistent UI presentation across reports.

**Source System:** Analytic Database (dbo.DimAssignmentStatus)

**Row Count:** 8 statuses (static set)

**Refresh Type:** Full refresh (small static table)

**Multi-Tenant:** No (statuses standardized across all tenants)

---

## Table Specifications

| Property | Value |
|----------|-------|
| Lineage Tag | c1f84207-4cf4-481d-9abe-4ef91e52f9d3 |
| Query Group | Dimensions |
| Partitions | 1 (Dim_Assignment_Status) |
| Relationships | Inbound from Assignments (many-to-one) |
| Calculated Columns | 2 (Colour, Status_Addtional) |
| Hierarchies | 0 |

---

## Column Specifications

| Column Name | Data Type | Description | Source Column | Key | Sort By | Nullable |
|-------------|-----------|-------------|---------------|-----|---------|----------|
| **Id** | int64 | Numeric status identifier (sort key) | Id | PK | - | No |
| **Status_Name** | string | Status display name | StatusName | - | Id | No |
| **Colour** | string | Hex color code for visualization (calculated) | - | - | - | No |
| **Status_Addtional** | string | Alternative status label (calculated) | - | - | - | Yes |

**Note:** Status_Name is sorted by Id column, ensuring consistent ordering (Draft → Active → Completed → Finalised → Archived → Cancelled)

---

## Calculated Columns

### Colour

**Purpose:** Assign hex color codes for status visualization in reports

**Data Type:** String (hex color code)

**DAX Expression:**
```dax
Colour = 
SWITCH(
    Dim_Assignment_Status[Id],
    1, "#39AAE3",  // Draft - Light Blue
    2, "#FFD073",  // Active - Yellow
    3, "#36DD38",  // Completed - Green
    4, "#0580A1",  // Finalised - Dark Teal
    5, "#B3B3B3",  // Archived - Gray
    6, "#F0390C",  // Cancelled - Red
    7, "#2D644C",  // Status 7 - Dark Green
    8, "#012435",  // Status 8 - Very Dark Blue
    "#000000"      // Default - Black
)
```

**Color Mapping:**
| Id | Status | Color | Hex Code | Visual Context |
|----|--------|-------|----------|----------------|
| 1 | Draft | Light Blue | #39AAE3 | In preparation |
| 2 | Active | Yellow | #FFD073 | In progress |
| 3 | Completed | Green | #36DD38 | Finished successfully |
| 4 | Finalised | Dark Teal | #0580A1 | Locked/approved |
| 5 | Archived | Gray | #B3B3B3 | Historical |
| 6 | Cancelled | Red | #F0390C | Stopped/rejected |
| 7 | (Reserved) | Dark Green | #2D644C | Future status |
| 8 | (Reserved) | Very Dark Blue | #012435 | Future status |

**Use Case:** Provides consistent color-coding across all reports for status indicators, cards, and conditional formatting

---

### Status_Addtional

**Purpose:** Alternative label mapping (currently maps Finalised → Completed)

**Data Type:** String

**DAX Expression:**
```dax
Status_Addtional = 
IF(
    Dim_Assignment_Status[Status_Name] = "Finalised",
    "Completed",
    Dim_Assignment_Status[Status_Name]
)
```

**Logic:**
- If Status_Name = "Finalised", returns "Completed"
- Otherwise returns Status_Name unchanged

**Use Case:** Provides alternative labeling for reports where "Finalised" and "Completed" should be treated as equivalent (e.g., user-facing reports may prefer "Completed" terminology over technical "Finalised")

---

## Relationships

### Inbound Relationships

**From Assignments**
- **Type:** Many-to-one
- **From Column:** Assignments[Status_Id]
- **To Column:** Dim_Assignment_Status[Id]
- **Cardinality:** Many:1
- **Cross-filter Direction:** Single
- **Status:** Active
- **Purpose:** Links assignments to their current status

---

## Power Query M Source

```m
let
    Source = Sql.Database(#"SQL Server", #"Analytic Database", [CreateNavigationProperties=false, MultiSubnetFailover=true]),
    Table = Source{[Schema="dbo",Item="DimAssignmentStatus"]}[Data],
    #"Renamed Columns" = Table.RenameColumns(Table,{
        {"StatusName", "Status_Name"}
    })
in
    #"Renamed Columns"
```

**Key Transformations:**
1. SQL database connection with MultiSubnetFailover
2. Column renaming (StatusName → Status_Name)
3. No filtering (all statuses included)

---

## DAX Query Patterns

### Status list with counts

```dax
EVALUATE
ADDCOLUMNS(
    Dim_Assignment_Status,
    "Status", [Status_Name],
    "Color", [Colour],
    "Alternative Label", [Status_Addtional],
    "Assignment Count", COUNTROWS(RELATEDTABLE(Assignments))
)
ORDER BY [Id]
```

### Active assignment statuses only

```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Dim_Assignment_Status,
        "Status", [Status_Name],
        "Color", [Colour]
    ),
    [Id] IN {2, 3}  // Active and Completed
)
```

### Status distribution

```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_Assignment_Status[Status_Name],
    Dim_Assignment_Status[Colour],
    "Total Assignments", COUNTROWS(Assignments),
    "Pct of Total", 
        DIVIDE(
            COUNTROWS(Assignments),
            CALCULATE(COUNTROWS(Assignments), ALL(Dim_Assignment_Status)),
            0
        )
)
ORDER BY [Total Assignments] DESC
```

### Color-coded status card measure

```dax
Status Color = 
SELECTEDVALUE(Dim_Assignment_Status[Colour], "#000000")

// Use in card visual with conditional formatting
// to display status with matching background color
```

---

## Data Model Pattern

**Pattern:** Small Lookup Dimension with Calculated Styling

**Characteristics:**
- **Static Data**: Fixed set of 8 status values
- **Sorted Display**: Status_Name sorted by Id ensures lifecycle order
- **Calculated Styling**: Colour column provides hex codes for visualization
- **Alternative Labels**: Status_Addtional enables flexible labeling

**Lifecycle Flow:**
```
Draft (1) → Active (2) → Completed (3) → Finalised (4) → Archived (5)
                              ↓
                        Cancelled (6)
```

**Usage in Reports:**
1. **Status Filters**: Enable users to filter by assignment lifecycle stage
2. **Color Coding**: Use Colour column for conditional formatting (cards, tables, charts)
3. **Status Cards**: Display current assignment counts by status with color backgrounds
4. **Progress Tracking**: Track assignments moving through lifecycle stages

**Visual Integration:**
The Colour column enables consistent color-coding without hardcoding colors in report measures or visuals. Changes to color scheme only require updating the SWITCH statement in the calculated column.

---

## Related Documentation

### ERD Documents
- [ERD #1: Assignment Core Model](../ERDs/ERD_01_Assignment_Core.md) - Documents assignment lifecycle

### Related Tables
- **Assignments** - Main fact table using Status_Id FK
- **Dim_Assignment_Categories** - Related assignment classification dimension

### Other Documentation
- [ERD_Overview.md](../ERD_Overview.md) - Architecture patterns
- [Assignments.md](Assignments.md) - Assignments fact table with status relationships

---

## Change History

| Date | Change Description | Modified By |
|------|-------------------|-------------|
| 2025-11-18 | Initial table documentation created from TMDL metadata | AI Documentation Generator |

---

## Notes

**Color Consistency**: The Colour column ensures consistent color-coding across all reports. Updates to color scheme require only updating the SWITCH statement, not individual report measures.

**Status_Addtional Purpose**: Currently maps "Finalised" to "Completed" for user-facing reports. This column can be extended to support additional alternative labels or localized status names without modifying the primary Status_Name column.

**Statuses 7-8**: Reserved for future status types. Colors assigned but Status_Name values not yet defined in source system.

**Sort Order**: Status_Name sorted by Id ensures Draft → Active → Completed → Finalised → Archived → Cancelled ordering in slicers and visuals, reflecting the logical assignment lifecycle progression.
