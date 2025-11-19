# Dim_Template_Groups

**Table Type:** Dimension Table

**Purpose:** Template group definitions for organizing templates into logical categories. Enables many-to-many grouping of templates.

**Last Updated:** November 19, 2025

---

## Overview

Dim_Template_Groups provides categorical groupings for work templates, enabling templates to be organized into logical collections such as "Safety Inspections", "Maintenance Procedures", "Environmental Checks", etc. A template can belong to multiple groups simultaneously through the Template_Group_WorkTemplates bridge table.

---

## Columns

| Column Name | Data Type | Description | Key |
|------------|-----------|-------------|-----|
| **Template_Group_Id** | String | Template group identifier | PK |
| **Tenant_Id** | String | Tenant identifier for multi-tenancy | - |
| **Template_Group_Name** | String | Group name for display | - |
| **Template_Group_Identifier** | String | Internal group identifier | - |
| **Created_Date** | DateTime | Creation timestamp | - |
| **Last_Updated** | DateTime | Last update timestamp | - |

---

## Relationships

**To Template_Group_WorkTemplates** (One-to-Many)
- **Relationship ID:** AutoDetected_30018288-cc51-408c-9930-4c0e08af691a
- **From:** Dim_Template_Groups[Template_Group_Id]
- **To:** Template_Group_WorkTemplates[Template_Group_Id]
- **Purpose:** Links groups to bridge table for many-to-many template membership

---

## Data Source

**Source Type:** SQL Query  
**Source Table:** DimTemplateGroups

```sql
SELECT 
    TemplateGroupId AS Template_Group_Id,
    TenantId AS Tenant_Id,
    TemplateGroupName AS Template_Group_Name,
    TemplateGroupIdentifier AS Template_Group_Identifier,
    CreatedDate AS Created_Date,
    LastUpdated AS Last_Updated
FROM DimTemplateGroups
WHERE ([Tenant filtering logic])
```

---

## Common DAX Patterns

### Templates per Group
```dax
Templates in Group = 
CALCULATE(
    DISTINCTCOUNT(Template_Group_WorkTemplates[Template_Link]),
    ALLEXCEPT(
        Template_Group_WorkTemplates,
        Template_Group_WorkTemplates[Template_Group_Id]
    )
)
```

### List All Groups
```dax
Template Group List = 
CONCATENATEX(
    Dim_Template_Groups,
    Dim_Template_Groups[Template_Group_Name],
    ", ",
    Dim_Template_Groups[Template_Group_Name],
    ASC
)
```

### Groups with Most Templates
```dax
EVALUATE
ADDCOLUMNS(
    Dim_Template_Groups,
    "Template Count", CALCULATE(COUNTROWS(Template_Group_WorkTemplates))
)
ORDER BY [Template Count] DESC
```

---

## Example Usage

**Template Groups:**
- "Safety Inspections" (Group_Id: abc123)
- "Daily Maintenance" (Group_Id: def456)
- "Environmental Compliance" (Group_Id: ghi789)

**Template Assignments (via bridge table):**
- "Truck Pre-Start" → Safety Inspections, Daily Maintenance
- "Spill Response" → Safety Inspections, Environmental Compliance
- "Oil Change" → Daily Maintenance

This many-to-many structure enables flexible template categorization for filtering and navigation in reports and mobile apps.

---

## Related Tables

### Bridge Table
- **Template_Group_WorkTemplates** - Many-to-many mapping to templates

### Templates
- **Dim_Published_Worktemplates** - Published work templates
- **Dim_WorkTemplates** - Full work template catalog

---

## Data Model Position

**Related ERDs:** ERD #6 (Templates, Fragments & Configuration)  
**Model Layer:** Dimension

---

## Change History

| Date | Change Description | Modified By |
|------|-------------------|-------------|
| 2025-11-19 | Initial table documentation created | AI Documentation Generator |
