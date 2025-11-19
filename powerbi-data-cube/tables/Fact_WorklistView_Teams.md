# Fact_WorklistView_Teams

**Table Type:** Fact Table (Access Control)

**Purpose:** Mapping of worklist views to teams for access control. Defines which teams can access which worklist views.

**Last Updated:** November 19, 2025

---

## Overview

Fact_WorklistView_Teams is an access control fact table that defines team-based permissions for worklist views. It enables sharing of custom worklist views with specific teams, creating team-scoped filtered views of assignments without making them tenant-wide.

---

## Columns

| Column Name | Data Type | Description | Key |
|------------|-----------|-------------|-----|
| **View_Id** | String | Worklist view identifier | FK |
| **Team_Id** | String | Team identifier | FK |
| **Team** | String | Team name (calculated via LOOKUPVALUE) | - |

---

## Calculated Columns

### Team (LOOKUPVALUE)
```dax
Team = LOOKUPVALUE(
    Dim_Teams[Name], 
    Dim_Teams[Team_Id], 
    Fact_WorklistView_Teams[Team_Id]
)
```

**Purpose:** Resolves team name without direct relationship to Dim_Teams

**Note:** No formal relationship exists between this table and Dim_Teams; LOOKUPVALUE provides the resolution at query time.

---

## Relationships

**To Dim_Worklist_Views** (Many-to-One)
- **Relationship ID:** 6392ffd9-4cea-6eac-3030-0c52e2ef0ebe
- **From:** Fact_WorklistView_Teams[View_Id]
- **To:** Dim_Worklist_Views[Id]
- **Purpose:** Links team access records to worklist view definitions

---

## Data Source

**Source Type:** SQL Query  
**Source Table:** FactWorklistViewTeams

```sql
SELECT 
    ViewId AS View_Id,
    TeamId AS Team_Id
FROM FactWorklistViewTeams
WHERE ([Tenant filtering logic])
```

**Note:** TenantId column exists in source but is removed during Power Query transformation (access control implied through view ownership).

---

## Access Control Logic

### Three View Sharing Modes

**1. Private View**
- Dim_Worklist_Views.Is_Shared_With_Tenant = FALSE
- No rows in Fact_WorklistView_Teams
- **Access:** Only view creator

**2. Team-Shared View**
- Dim_Worklist_Views.Is_Shared_With_Tenant = FALSE
- One or more rows in Fact_WorklistView_Teams
- **Access:** Creator + specified teams

**3. Tenant-Shared View**
- Dim_Worklist_Views.Is_Shared_With_Tenant = TRUE
- Fact_WorklistView_Teams rows optional (may exist but not enforced)
- **Access:** All users in tenant

---

## Common DAX Patterns

### Views Accessible by Team
```dax
Team's Worklist Views = 
CALCULATE(
    COUNTROWS(Dim_Worklist_Views),
    Fact_WorklistView_Teams[Team] = "Maintenance Team"
)
```

### Teams per View
```dax
Teams with Access = 
CALCULATE(
    DISTINCTCOUNT(Fact_WorklistView_Teams[Team_Id]),
    ALLEXCEPT(
        Fact_WorklistView_Teams,
        Fact_WorklistView_Teams[View_Id]
    )
)
```

### View Access Matrix
```dax
EVALUATE
SELECTCOLUMNS(
    Fact_WorklistView_Teams,
    "View", RELATED(Dim_Worklist_Views[Name]),
    "Team", [Team],
    "Creator", RELATED(Dim_User_Reference[Full_Name])
)
ORDER BY [View], [Team]
```

### Views with Multiple Teams
```dax
Multi-Team Views = 
CALCULATE(
    DISTINCTCOUNT(Fact_WorklistView_Teams[View_Id]),
    FILTER(
        VALUES(Fact_WorklistView_Teams[View_Id]),
        CALCULATE(DISTINCTCOUNT(Fact_WorklistView_Teams[Team_Id])) > 1
    )
)
```

---

## Related Tables

### Configuration
- **Dim_Worklist_Views** - View definitions with sharing flags

### Team Information (No Direct Relationship)
- **Dim_Teams** - Team names resolved via LOOKUPVALUE

---

## Data Model Position

**Related ERDs:** ERD #6 (Templates, Fragments & Configuration), ERD #7 (Fact Tables & Audit)  
**Model Layer:** Fact Table (Access Control)

---

## Change History

| Date | Change Description | Modified By |
|------|-------------------|-------------|
| 2025-11-19 | Initial table documentation created | AI Documentation Generator |

---

## Additional Notes

### Why LOOKUPVALUE Instead of Relationship?
The Team calculated column uses LOOKUPVALUE rather than a formal relationship to Dim_Teams. Reasons:
- **Simplicity:** Avoids additional relationship complexity
- **Query-Time Resolution:** Team name retrieved only when needed
- **Flexibility:** Doesn't impact model relationship cardinality
- **Performance:** Small table size makes LOOKUPVALUE performant

### Access Control Enforcement
This table provides metadata for access control but does not enforce RLS directly. The Obzervr application uses this data to filter available views in the UI. For report-level RLS, use RLS_Users and related security tables.
