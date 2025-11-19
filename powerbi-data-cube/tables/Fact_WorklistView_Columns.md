# Fact_WorklistView_Columns

**Table Type:** Fact Table (Configuration)

**Purpose:** Column configuration metadata for worklist views. Defines display order, grouping, sorting, and width settings for columns in each worklist view.

**Last Updated:** November 19, 2025

---

## Overview

Fact_WorklistView_Columns stores the column display configuration for custom worklist views in Obzervr. Each row defines how a single column should be rendered in a worklist view, including its display order, grouping behavior, sort priority, sort direction, and pixel width.

---

## Columns

| Column Name | Data Type | Description |
|------------|-----------|-------------|
| **Id** | String | Column configuration ID (PK) |
| **View_Id** | String | Worklist view identifier (FK) |
| **Tenant_Id** | String | Tenant identifier |
| **Name** | String | Column display name |
| **Index** | Integer | Display order index (ascending) |
| **Group_Index** | Integer | Grouping index (for grouped columns) |
| **Sort_Index** | Integer | Sort priority (lower = higher priority) |
| **Sort_Order** | String | Sort direction or expression |
| **Width** | Integer | Column width in pixels |
| **Created_Date** | DateTime | Creation timestamp |
| **Last_Updated** | DateTime | Last update timestamp |

---

## Column Attributes

### Index (Display Order)
- **Purpose:** Controls left-to-right column order in view
- **Values:** 0, 1, 2, 3, ... (ascending)
- **Example:** Index=0 is leftmost column, Index=5 is sixth column

### Group_Index (Column Grouping)
- **Purpose:** Groups related columns together
- **Values:** Integer grouping identifier
- **Example:** Group_Index=1 might group "Assignment Info" columns, Group_Index=2 might group "Date" columns
- **Usage:** UI rendering hint for visual column grouping

### Sort_Index (Sort Priority)
- **Purpose:** Defines multi-column sort order
- **Values:** 0, 1, 2, ... (lower = higher priority)
- **Example:** Sort_Index=0 is primary sort, Sort_Index=1 is secondary sort
- **Usage:** Applied when view loads to pre-sort data

### Sort_Order
- **Purpose:** Defines sort direction or custom sort expression
- **Values:** "ASC" (ascending), "DESC" (descending), or custom expression
- **Example:** "ASC", "DESC", "CASE WHEN Status = 'Open' THEN 0 ELSE 1 END"

### Width
- **Purpose:** Column width in pixels
- **Values:** Integer pixel count
- **Typical Values:** 100-300 pixels depending on column content
- **Usage:** Sets initial column width in UI

---

## Relationships

**To Dim_Worklist_Views** (Many-to-One)
- **Relationship ID:** 4fc32637-72c9-eb68-61df-2301c44f20f7
- **From:** Fact_WorklistView_Columns[View_Id]
- **To:** Dim_Worklist_Views[Id]
- **Purpose:** Links column configurations to worklist view definitions

---

## Data Source

**Source Type:** SQL Query  
**Source Table:** FactWorklistViewColumns

```sql
SELECT 
    Id,
    ViewId AS View_Id,
    TenantId AS Tenant_Id,
    Name,
    [Index],
    GroupIndex AS Group_Index,
    SortIndex AS Sort_Index,
    SortOrder AS Sort_Order,
    Width,
    CreatedDate AS Created_Date,
    LastUpdated AS Last_Updated
FROM FactWorklistViewColumns
WHERE ([Tenant filtering logic])
```

---

## Common DAX Patterns

### Columns per View
```dax
Columns in View = 
CALCULATE(
    COUNTROWS(Fact_WorklistView_Columns),
    ALLEXCEPT(
        Fact_WorklistView_Columns,
        Fact_WorklistView_Columns[View_Id]
    )
)
```

### Column Configuration for Specific View
```dax
EVALUATE
SELECTCOLUMNS(
    FILTER(
        Fact_WorklistView_Columns,
        RELATED(Dim_Worklist_Views[Name]) = "My Active Assignments"
    ),
    "Column Name", [Name],
    "Display Order", [Index],
    "Group", [Group_Index],
    "Sort Priority", [Sort_Index],
    "Sort Direction", [Sort_Order],
    "Width (px)", [Width]
)
ORDER BY [Display Order]
```

### Average Column Width
```dax
Avg Column Width = 
AVERAGE(Fact_WorklistView_Columns[Width])
```

### Sorted Columns Count
```dax
Sorted Columns = 
CALCULATE(
    COUNTROWS(Fact_WorklistView_Columns),
    NOT ISBLANK(Fact_WorklistView_Columns[Sort_Index])
)
```

---

## Example Configuration

**View:** "Team Work Orders"

| Name | Index | Group_Index | Sort_Index | Sort_Order | Width |
|------|-------|-------------|------------|------------|-------|
| Assignment Number | 0 | 1 | NULL | NULL | 150 |
| Status | 1 | 1 | 0 | ASC | 100 |
| Priority | 2 | 1 | 1 | DESC | 80 |
| Assigned To | 3 | 2 | NULL | NULL | 200 |
| Due Date | 4 | 3 | 2 | ASC | 120 |
| Location | 5 | 4 | NULL | NULL | 250 |

**Result:**
- 6 columns displayed left to right by Index
- Grouped: Assignment Info (1), People (2), Dates (3), Location (4)
- Sorted by: Status ASC (primary), then Priority DESC (secondary), then Due Date ASC (tertiary)
- Column widths set in pixels

---

## Related Tables

### Configuration
- **Dim_Worklist_Views** - View definitions
- **Fact_WorklistView_Teams** - Team access control for views

---

## Data Model Position

**Related ERDs:** ERD #6 (Templates, Fragments & Configuration), ERD #7 (Fact Tables & Audit)  
**Model Layer:** Fact Table (Configuration)

---

## Change History

| Date | Change Description | Modified By |
|------|-------------------|-------------|
| 2025-11-19 | Initial table documentation created | AI Documentation Generator |

---

## Additional Notes

### Column Configuration Flexibility
This table enables highly flexible worklist views:
- **Dynamic Columns:** Users can show/hide columns
- **Custom Ordering:** Drag-and-drop reordering reflected in Index
- **Multi-Level Sorting:** Up to N columns for complex sorting
- **Responsive Width:** User-adjusted widths persisted

### UI Integration
The Obzervr application reads this configuration to render worklist views with user preferences applied. Changes made in the UI are persisted back to this table via API calls.

### Null Handling
- **Sort_Index NULL:** Column not included in sorting
- **Group_Index NULL or 0:** Column not grouped
- **Sort_Order NULL:** Default ascending sort if Sort_Index is set
