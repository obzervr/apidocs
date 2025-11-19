# Template_Group_WorkTemplates

**Table Type:** Bridge Table (Many-to-Many)

**Purpose:** Many-to-many bridge table mapping template groups to work templates. Enables templates to belong to multiple groups.

**Last Updated:** November 19, 2025

---

## Overview

Template_Group_WorkTemplates is a bridge table that implements the many-to-many relationship between template groups and work templates. It allows a single template to be categorized into multiple groups, and a group to contain multiple templates, providing flexible organizational structure for template management.

---

## Columns

| Column Name | Data Type | Description | Key |
|------------|-----------|-------------|-----|
| **Id** | String | Record identifier | PK |
| **Template_Group_Id** | String | Template group reference | FK |
| **Template_Link** | String | Template reference link | FK |
| **Tenant_Id** | String | Tenant identifier for multi-tenancy | - |
| **Created_Date** | DateTime | Creation timestamp | - |
| **Last_Updated** | DateTime | Last update timestamp | - |

---

## Relationships

**To Dim_Template_Groups** (Many-to-One)
- **Relationship ID:** AutoDetected_30018288-cc51-408c-9930-4c0e08af691a
- **From:** Template_Group_WorkTemplates[Template_Group_Id]
- **To:** Dim_Template_Groups[Template_Group_Id]
- **Purpose:** Links to template group definitions

**To Dim_Published_Worktemplates** (Many-to-One)
- **Relationship ID:** 292b26b0-33cf-5f9e-0ae0-2115cd7ac91a (from Dim_WorkTemplates)
- **Relationship ID:** 302 (from calculated table)
- **From:** Template_Group_WorkTemplates[Template_Link]
- **To:** Dim_Published_Worktemplates[Template_Link]
- **Purpose:** Links to published work templates

**To Dim_WorkTemplates** (Many-to-One)
- **From:** Template_Group_WorkTemplates[Template_Link]
- **To:** Dim_WorkTemplates[Template_Link]
- **Purpose:** Links to full work template catalog

---

## Data Source

**Source Type:** SQL Query  
**Source Table/View:** VW_TemplateGroupWorkTemplates

```sql
SELECT 
    Id,
    TemplateGroupId AS Template_Group_Id,
    TemplateLink AS Template_Link,
    TenantId AS Tenant_Id,
    CreatedDate AS Created_Date,
    LastUpdated AS Last_Updated
FROM VW_TemplateGroupWorkTemplates
WHERE ([Tenant filtering logic])
```

---

## Common DAX Patterns

### Templates by Group
```dax
EVALUATE
SELECTCOLUMNS(
    Template_Group_WorkTemplates,
    "Group", RELATED(Dim_Template_Groups[Template_Group_Name]),
    "Template", RELATED(Dim_Published_Worktemplates[Name]),
    "Template Link", [Template_Link]
)
ORDER BY [Group], [Template]
```

### Count Groups per Template
```dax
Groups per Template = 
CALCULATE(
    DISTINCTCOUNT(Template_Group_WorkTemplates[Template_Group_Id]),
    ALLEXCEPT(
        Template_Group_WorkTemplates,
        Template_Group_WorkTemplates[Template_Link]
    )
)
```

### Templates in Multiple Groups
```dax
Templates in Multiple Groups = 
CALCULATE(
    DISTINCTCOUNT(Template_Group_WorkTemplates[Template_Link]),
    FILTER(
        VALUES(Template_Group_WorkTemplates[Template_Link]),
        CALCULATE(
            DISTINCTCOUNT(Template_Group_WorkTemplates[Template_Group_Id])
        ) > 1
    )
)
```

### Group Membership Matrix
```dax
EVALUATE
SUMMARIZE(
    Template_Group_WorkTemplates,
    RELATED(Dim_Template_Groups[Template_Group_Name]),
    RELATED(Dim_Published_Worktemplates[Name]),
    "Assigned Date", [Created_Date]
)
```

---

## Example Data

| Id | Template_Group_Id | Template_Link | Tenant_Id |
|----|------------------|---------------|-----------|
| 001 | grp_safety | tmpl_prestart | tenant1 |
| 002 | grp_maintenance | tmpl_prestart | tenant1 |
| 003 | grp_safety | tmpl_spill | tenant1 |
| 004 | grp_environmental | tmpl_spill | tenant1 |

This shows "Pre-Start" template belongs to both Safety and Maintenance groups, while "Spill Response" belongs to both Safety and Environmental groups.

---

## Related Tables

### Dimension Tables
- **Dim_Template_Groups** - Template group definitions
- **Dim_Published_Worktemplates** - Published work templates
- **Dim_WorkTemplates** - Full work template catalog

---

## Data Model Position

**Related ERDs:** ERD #6 (Templates, Fragments & Configuration)  
**Model Layer:** Bridge Table (Many-to-Many)

---

## Change History

| Date | Change Description | Modified By |
|------|-------------------|-------------|
| 2025-11-19 | Initial table documentation created | AI Documentation Generator |
