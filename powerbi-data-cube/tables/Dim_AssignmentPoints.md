# Dim_AssignmentPoints Table

**Last Updated**: November 18, 2025

## Overview

Assignment point master list with hierarchy, coordinates, and parent links. This dimension table provides a 9-level hierarchical structure for organizing physical locations and assets where work is performed. Includes geospatial coordinates and parent-child relationships for drill-down analysis.

## Table Specifications

| Property | Value |
|----------|-------|
| **Table Name** | Dim_AssignmentPoints |
| **Table Type** | Dimension Table (Hierarchical) |
| **Lineage Tag** | 0e8ff35e-5a26-4b74-ad25-187e467d8782 |
| **Refresh Policy** | Incremental Refresh (20 year rolling window, 1 year increments) |
| **Incremental Column** | Created_Date |
| **Polling Expression** | Last_Updated (with NULL handling) |
| **Source View** | VW_AssignmentPoints (Analytic Database) |
| **Hierarchy** | Point Hierarchy (9 levels) |
| **Related ERDs** | [ERD #1: Assignment Core](../ERDs/ERD_01_Assignment_Core.md) |

---

## Columns

### Source Columns (13)

| Column Name | Data Type | Description | Format | Aggregation |
|-------------|-----------|-------------|--------|-------------|
| **Id** | string | Unique identifier for the assignment point | - | None |
| **Point_Name** | string | Descriptive name of the point | - | None |
| **AssignmentPoint_Type_Name** | string | Type classification of the assignment point | - | None |
| **Point_Id** | string | Business identifier for the point | - | None |
| **AP_Latitude** | double | Latitude coordinate for the point location | General Number | Sum |
| **AP_Longitude** | double | Longitude coordinate for the point location | General Number | Sum |
| **Parent_Id** | string | Identifier of the parent point in the hierarchy | - | None |
| **IsDeleted** | boolean | Soft delete flag indicating if point is inactive | TRUE/FALSE | None |
| **SubSite_Id** | string | Identifier linking to sub-site dimension | - | None |
| **Created_Date** | datetime | Timestamp when the point was created | General Date | None |
| **Last_Updated** | datetime | Timestamp of last modification | General Date | None |
| **Tenant_Id** | string | Tenant identifier for multi-tenancy | - | None |

---

## Calculated Columns

### Point
**Description**: Display name combining Point_Id and Point_Name based on configuration

**DAX Expression**:
```dax
IF(
    Dim_AssignmentPoints[Point_Id] = Dim_AssignmentPoints[Point_Name],
    Dim_AssignmentPoints[Point_Id],
    IF(
        CONTAINS(
            UsePointNameOnlyAsPoint, 
            UsePointNameOnlyAsPoint[UsePointNameOnlyAsPoint], 
            TRUE()
        ),
        Dim_AssignmentPoints[Point_Name],
        Dim_AssignmentPoints[Point_Id] & " " & Dim_AssignmentPoints[Point_Name]
    )
)
```

**Purpose**: Provides flexible display format - can show ID only, name only, or combined based on tenant configuration.

---

### Parent Hierarchy (P1 through P9)

**P1** - Direct parent
```dax
Dim_AssignmentPoints[Parent_Id]
```

**P2** - Grandparent
```dax
LOOKUPVALUE(
    Dim_AssignmentPoints[Parent_Id], 
    Dim_AssignmentPoints[Id], 
    Dim_AssignmentPoints[P1]
)
```

**P3 through P9** - Continue pattern up to 9 levels
```dax
LOOKUPVALUE(
    Dim_AssignmentPoints[Parent_Id], 
    Dim_AssignmentPoints[Id], 
    Dim_AssignmentPoints[P{n-1}]
)
```

**Purpose**: Materializes ancestor path for performance. Enables fast hierarchy navigation without recursive queries.

---

### Max Level
**Description**: Determines the depth of the point in the hierarchy (1-9)

**Format**: 0

**DAX Expression**:
```dax
IF(
    ISBLANK(Dim_AssignmentPoints[P1]), 1,
    IF(ISBLANK(Dim_AssignmentPoints[P2]), 2,
        IF(ISBLANK(Dim_AssignmentPoints[P3]), 3,
            IF(ISBLANK(Dim_AssignmentPoints[P4]), 4,
                IF(ISBLANK(Dim_AssignmentPoints[P5]), 5,
                    IF(ISBLANK(Dim_AssignmentPoints[P6]), 6,
                        IF(ISBLANK(Dim_AssignmentPoints[P7]), 7,
                            IF(ISBLANK(Dim_AssignmentPoints[P8]), 8, 9)
                        )
                    )
                )
            )
        )
    )
)
```

**Purpose**: Identifies how deep the point sits in the hierarchy. Used for dynamic level calculations.

---

### Level 1 - Name through Level 9 - Name
**Description**: Shows the point name at each hierarchy level, normalized so Level 1 is always the top

**Example DAX (Level 1)**:
```dax
SWITCH(
    Dim_AssignmentPoints[Max Level],
    1, Dim_AssignmentPoints[Point],
    2, LOOKUPVALUE(Dim_AssignmentPoints[Point], Dim_AssignmentPoints[Id], Dim_AssignmentPoints[P1]),
    3, LOOKUPVALUE(Dim_AssignmentPoints[Point], Dim_AssignmentPoints[Id], Dim_AssignmentPoints[P2]),
    4, LOOKUPVALUE(Dim_AssignmentPoints[Point], Dim_AssignmentPoints[Id], Dim_AssignmentPoints[P3]),
    5, LOOKUPVALUE(Dim_AssignmentPoints[Point], Dim_AssignmentPoints[Id], Dim_AssignmentPoints[P4]),
    6, LOOKUPVALUE(Dim_AssignmentPoints[Point], Dim_AssignmentPoints[Id], Dim_AssignmentPoints[P5]),
    7, LOOKUPVALUE(Dim_AssignmentPoints[Point], Dim_AssignmentPoints[Id], Dim_AssignmentPoints[P6]),
    8, LOOKUPVALUE(Dim_AssignmentPoints[Point], Dim_AssignmentPoints[Id], Dim_AssignmentPoints[P7]),
    9, LOOKUPVALUE(Dim_AssignmentPoints[Point], Dim_AssignmentPoints[Id], Dim_AssignmentPoints[P8])
)
```

**Level 2 through Level 9** show "N/A" if the point doesn't reach that depth.

**Purpose**: Creates a normalized hierarchy where every point has all levels populated. Top-level points repeat their name, lower levels show "N/A". Enables consistent hierarchy visual in Power BI.

---

### Enable_TSFM_AssignmentPoint
**Description**: Flag to enable/disable TimeSeries Field Measurement filtering

**Format**: TRUE/FALSE

**Value**: Always TRUE()

**Hidden**: Yes

**Purpose**: Configuration flag used in relationships to control TSFM-level filtering.

---

## Hierarchy

### Point Hierarchy (9 levels)

The table includes a pre-built hierarchy with 9 levels:

1. **Level 1** (Top) - Site/Region
2. **Level 2** - Area/Zone  
3. **Level 3** - Department/Section
4. **Level 4** - System/Unit
5. **Level 5** - Equipment/Asset
6. **Level 6** - Sub-component
7. **Level 7** - Part/Element
8. **Level 8** - Sub-part
9. **Level 9** (Bottom) - Specific location

**Usage**: Drag the entire hierarchy onto visuals for automatic drill-down capability.

---

## Relationships

### Active Relationships

| To Table | From Column | To Column | Cardinality | Description |
|----------|-------------|-----------|-------------|-------------|
| **Dim_Subsites** | SubSite_Id | Id | Many-to-One | Links point to organizational subsite |
| **Assignments** | Id | AssignmentPoint_Id | One-to-Many | Points have multiple assignments |
| **Dim_Is_AssignmentPoint_Active** | IsDeleted | Value | Many-to-One | Active/inactive status lookup |
| **EnableTSFMAssignmentPoint** | Enable_TSFM_AssignmentPoint | EnableTSFMAssignmentPoint | Many-to-One | TSFM filter control |

### Inactive Relationships

| To Table | From Column | Purpose |
|----------|-------------|---------|
| **Dim_Tenant** | Tenant_Id | Can be activated for cross-tenant analysis |

### Bridge Tables

| Bridge Table | Purpose |
|--------------|---------|
| **AssignmentPoint_Attributes** | EAV pattern for custom attributes (bidirectional) |
| **AssignmentPoint_Reference_TSFM** | Links points to field measurements |
| **Fact_Rosters** | Roster assignments to points |

---

## Common DAX Query Patterns

### Count Assignments by Hierarchy Level 1
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_AssignmentPoints[Level 1 - Name],
    "Assignment Count", 
        CALCULATE(COUNTROWS(Assignments)),
    "Active Points",
        CALCULATE(
            COUNTROWS(Dim_AssignmentPoints),
            Dim_AssignmentPoints[IsDeleted] = FALSE
        )
)
ORDER BY [Assignment Count] DESC
```

### Find Points with Coordinates
```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Dim_AssignmentPoints,
        "Point", Dim_AssignmentPoints[Point],
        "Latitude", Dim_AssignmentPoints[AP_Latitude],
        "Longitude", Dim_AssignmentPoints[AP_Longitude],
        "Type", Dim_AssignmentPoints[AssignmentPoint_Type_Name]
    ),
    NOT(ISBLANK(Dim_AssignmentPoints[AP_Latitude])) &&
    NOT(ISBLANK(Dim_AssignmentPoints[AP_Longitude]))
)
```

### Hierarchy Depth Analysis
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_AssignmentPoints[Max Level],
    "Point Count", COUNTROWS(Dim_AssignmentPoints),
    "Avg Assignments", 
        AVERAGEX(
            Dim_AssignmentPoints,
            CALCULATE(COUNTROWS(Assignments))
        )
)
ORDER BY Dim_AssignmentPoints[Max Level]
```

### Find Orphaned Points (No Parent)
```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Dim_AssignmentPoints,
        "Point", Dim_AssignmentPoints[Point],
        "Point ID", Dim_AssignmentPoints[Point_Id],
        "Type", Dim_AssignmentPoints[AssignmentPoint_Type_Name],
        "Max Level", Dim_AssignmentPoints[Max Level]
    ),
    ISBLANK(Dim_AssignmentPoints[Parent_Id]) &&
    Dim_AssignmentPoints[IsDeleted] = FALSE
)
```

### Parent-Child Lineage
```dax
EVALUATE
SELECTCOLUMNS(
    Dim_AssignmentPoints,
    "Point", Dim_AssignmentPoints[Point],
    "Level 1", Dim_AssignmentPoints[Level 1 - Name],
    "Level 2", Dim_AssignmentPoints[Level 2 - Name],
    "Level 3", Dim_AssignmentPoints[Level 3 - Name],
    "Level 4", Dim_AssignmentPoints[Level 4 - Name],
    "Depth", Dim_AssignmentPoints[Max Level]
)
ORDER BY [Level 1], [Level 2], [Level 3], [Level 4]
```

---

## Data Model Pattern: Parent-Child Hierarchy

This table implements a **flattened parent-child hierarchy** pattern:

### Structure

**Flattened Hierarchy Implementation**:
- Pre-calculated ancestor path (P1-P9 columns)
- Built-in Power BI hierarchy support
- Fixed maximum depth of 9 levels
- Level columns normalize hierarchy for consistent visuals

### Hierarchy Behavior
- Top-level points (no parent) have NULL P1
- Each level column shows appropriate ancestor name
- Points below max depth show "N/A" for unused levels
- Enables consistent drill-down in Power BI matrix visuals

---

## Geospatial Features

### Coordinate Storage
- Latitude/Longitude stored as decimal degrees
- NULL values indicate non-geolocated points

### Usage in Power BI
```dax
// Create map visual with custom coordinates
EVALUATE
SELECTCOLUMNS(
    FILTER(
        Dim_AssignmentPoints,
        NOT(ISBLANK(Dim_AssignmentPoints[AP_Latitude]))
    ),
    "Location", Dim_AssignmentPoints[Point],
    "Lat", Dim_AssignmentPoints[AP_Latitude],
    "Long", Dim_AssignmentPoints[AP_Longitude],
    "Assignment Count", 
        CALCULATE(COUNTROWS(Assignments))
)
```

---

## Related Documentation

- **[ERD #1: Assignment Core Model](../ERDs/ERD_01_Assignment_Core.md)** - Core relationships
- **[Assignments](Assignments.md)** - Main fact table using this dimension
- **[AssignmentPoint_Attributes](AssignmentPoint_Attributes.md)** - Custom attributes (EAV)
- **[Dim_Subsites](Dim_Subsites.md)** - Organizational hierarchy
- **[Dim_Reference_AssignmentPoints](Dim_Reference_AssignmentPoints.md)** - Alternate point dimension for TSFM

---

## Change History

| Date | Change | Author |
|------|--------|--------|
| 2025-11-18 | Initial documentation generated from TMDL | AI Documentation Generator |

