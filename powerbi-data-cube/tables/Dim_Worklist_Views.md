# Dim_Worklist_Views

**Table Type:** Dimension Table

**Purpose:** User-created worklist view definitions with filters, pagination, and sharing settings. Enables customized work list displays per user and tenant.

**Last Updated:** November 19, 2025

---

## Overview

Dim_Worklist_Views stores configuration for customized work list views created by users in the Obzervr application. Each view defines filtering criteria, column display preferences, pagination settings, and sharing permissions, enabling personalized or team-shared assignment list layouts.

---

## Columns

| Column Name | Data Type | Description | Key |
|------------|-----------|-------------|-----|
| **Id** | String | View identifier | PK |
| **Tenant_Id** | String | Tenant identifier | FK |
| **Name** | String | View name | - |
| **Description** | String | View description | - |
| **Type** | String | View type classification | - |
| **Filter** | String | Serialized filter definition (JSON or similar) | - |
| **Page_Size** | Integer | Items per page for pagination | - |
| **Is_Shared_With_Tenant** | Boolean | Tenant-wide sharing flag | - |
| **Created_By** | String | Creator user ID | FK |
| **Updated_By** | String | Last updater user ID | - |
| **Created_Date** | DateTime | Creation timestamp | - |
| **Last_Updated** | DateTime | Last update timestamp | - |

---

## Relationships

**To Dim_User_Reference** (Many-to-One)
- **Relationship ID:** b769584b-bb21-57f9-ac05-43d77a7b7c78
- **From:** Dim_Worklist_Views[Created_By]
- **To:** Dim_User_Reference[User_Id]
- **Purpose:** Links views to creating user for ownership tracking

**To Dim_Tenant** (Many-to-One)
- **Relationship ID:** c3be2bf1-d131-e98f-cb35-cc64d24f4be3
- **From:** Dim_Worklist_Views[Tenant_Id]
- **To:** Dim_Tenant[Tenant_Id]
- **Purpose:** Multi-tenant isolation

**To Fact_WorklistView_Teams** (One-to-Many)
- **Relationship ID:** 6392ffd9-4cea-6eac-3030-0c52e2ef0ebe
- **From:** Dim_Worklist_Views[Id]
- **To:** Fact_WorklistView_Teams[View_Id]
- **Purpose:** Team access control configuration

**To Fact_WorklistView_Columns** (One-to-Many)
- **Relationship ID:** 4fc32637-72c9-eb68-61df-2301c44f20f7
- **From:** Dim_Worklist_Views[Id]
- **To:** Fact_WorklistView_Columns[View_Id]
- **Purpose:** Column display configuration

---

## Data Source

**Source Type:** SQL Query  
**Source Table:** DimWorklistViews

```sql
SELECT 
    Id,
    TenantId AS Tenant_Id,
    Name,
    Description,
    Type,
    Filter,
    PageSize AS Page_Size,
    IsSharedWithTenant AS Is_Shared_With_Tenant,
    CreatedBy AS Created_By,
    UpdatedBy AS Updated_By,
    CreatedDate AS Created_Date,
    LastUpdated AS Last_Updated
FROM DimWorklistViews
WHERE ([Tenant filtering logic])
```

---

## Key Attributes

### Filter Column
- **Format:** Serialized string (likely JSON)
- **Contains:** Filter predicates (e.g., status, date range, assigned user, location)
- **Example:** `{"status": ["Open", "In Progress"], "assignedTo": "user@example.com", "fromDate": ">= 2025-01-01"}`
- **Purpose:** Defines which assignments appear in the view

### Page_Size
- **Type:** Integer
- **Purpose:** Pagination control (number of items per page)
- **Typical Values:** 25, 50, 100, 200
- **Usage:** Mobile and web app pagination

### Is_Shared_With_Tenant
- **True:** View visible to all users in tenant
- **False:** View visible only to creator (private view)
- **Purpose:** Enables shared team views vs personal views

### Type
- **Purpose:** View categorization
- **Examples:** "My Assignments", "Team View", "Location View", "Status Dashboard"
- **Usage:** UI organization and filtering

---

## Common DAX Patterns

### Views by User
```dax
User's Worklist Views = 
CALCULATE(
    COUNTROWS(Dim_Worklist_Views),
    RELATED(Dim_User_Reference[Email]) = USERPRINCIPALNAME()
)
```

### Shared Views by Tenant
```dax
EVALUATE
SELECTCOLUMNS(
    FILTER(
        Dim_Worklist_Views, 
        [Is_Shared_With_Tenant] = TRUE
    ),
    "Tenant", RELATED(Dim_Tenant[Tenant_Name]),
    "View Name", [Name],
    "Creator", RELATED(Dim_User_Reference[Full_Name]),
    "Type", [Type]
)
```

### Views with Team Access
```dax
Team-Accessible Views = 
CALCULATE(
    COUNTROWS(Dim_Worklist_Views),
    NOT ISEMPTY(Fact_WorklistView_Teams)
)
```

### Views by Type
```dax
EVALUATE
SUMMARIZE(
    Dim_Worklist_Views,
    [Type],
    "View Count", COUNTROWS(Dim_Worklist_Views),
    "Shared Count", CALCULATE(
        COUNTROWS(Dim_Worklist_Views), 
        [Is_Shared_With_Tenant] = TRUE
    )
)
```

---

## Related Tables

### Configuration Tables
- **Fact_WorklistView_Teams** - Team access control for views
- **Fact_WorklistView_Columns** - Column configuration for views

### Reference Tables
- **Dim_User_Reference** - View creator user details
- **Dim_Tenant** - Multi-tenant isolation

---

## Data Model Position

**Related ERDs:** ERD #6 (Templates, Fragments & Configuration)  
**Model Layer:** Dimension

---

## Change History

| Date | Change Description | Modified By |
|------|-------------------|-------------|
| 2025-11-19 | Initial table documentation created | AI Documentation Generator |

---

## Additional Notes

### View Configuration Pattern
Each worklist view consists of three components:
1. **Dim_Worklist_Views:** Core view definition (this table)
2. **Fact_WorklistView_Teams:** Which teams can access the view
3. **Fact_WorklistView_Columns:** Which columns to display and how

### Sharing Logic
- **Private View:** Is_Shared_With_Tenant = FALSE, no teams in Fact_WorklistView_Teams
- **Shared View:** Is_Shared_With_Tenant = TRUE (all tenant users can access)
- **Team View:** Is_Shared_With_Tenant = FALSE, specific teams in Fact_WorklistView_Teams

### Filter Serialization
The Filter column contains serialized filter criteria that the Obzervr application deserializes to apply assignment filtering. This enables complex, multi-criteria views without schema changes.
