# Dim_Published_Worktemplates

**Table Type:** Calculated Dimension Table

**Purpose:** Published work template catalog with template names, identifiers, and links. Filtered from Dim_WorkTemplates to include only published templates (Is_Published = TRUE).

**Last Updated:** November 19, 2025

---

## Overview

Dim_Published_Worktemplates is a calculated dimension table that provides a focused view of published work templates available for use in assignments. It is derived from the full Dim_WorkTemplates table through DAX filtering, containing only templates that have been published and are ready for operational use. This separation enables cleaner reporting by excluding draft, archived, or unpublished templates from standard reports and slicers.

---

## Columns

| Column Name | Data Type | Description | Characteristics |
|------------|-----------|-------------|-----------------|
| **Template_Link** | String | Template reference link (Primary Key) | PK, Unique identifier |
| **Name** | String | Template name | Display-friendly |
| **Identifier** | String | Template identifier | Internal reference |
| **Id** | String | Template ID | System ID |
| **Fragment_Type** | String | Fragment type classification | FK to Dim_Fragment_Type |
| **Published_At** | DateTime | Publication timestamp (from Last_Updated) | Format: dd/MM/yyyy |
| **Template_URL** | String | Calculated: Tenant URL + /designer/template/ + Id | DataCategory: WebUrl |

---

## Calculated Columns

### Template_URL
```dax
Template_URL = 
MAX(Dim_Tenant[Tenant_URL]) & "/designer/template/" & Dim_Published_Worktemplates[Id]
```

**Purpose:** Generates deep link to template designer page in Obzervr application

**Example Output:** `https://acme.obzervr.com/designer/template/abc123def456`

**Usage:** Clickable links in reports for template navigation

**Data Category:** WebUrl (enables automatic hyperlink rendering in Power BI)

---

## Table Source (DAX Calculated Table)

```dax
Dim_Published_Worktemplates = 
VAR __DS0FilterTable = 
    TREATAS({TRUE}, 'Dim_WorkTemplates'[Is_Published])
RETURN
CALCULATETABLE(
    SUMMARIZE(
        'Dim_WorkTemplates',
        'Dim_WorkTemplates'[Name],
        'Dim_WorkTemplates'[Template_Link],
        'Dim_WorkTemplates'[Identifier],
        'Dim_WorkTemplates'[Id],
        'Dim_WorkTemplates'[Fragment_Type],
        'Dim_WorkTemplates'[Last_Updated]
    ),
    KEEPFILTERS(TREATAS({TRUE}, 'Dim_WorkTemplates'[Is_Published]))
)
```

**Key Transformations:**
1. Filter Dim_WorkTemplates where Is_Published = TRUE
2. Summarize to include only relevant columns
3. Rename Last_Updated to Published_At
4. Add calculated Template_URL column
5. Format Published_At as dd/MM/yyyy

**Benefits of Calculated Table:**
- Dynamic filtering (updates automatically when Dim_WorkTemplates refreshes)
- Reduced column set (only essential template information)
- No duplicate data storage (virtual table)
- Simplified report building (no need to filter Is_Published in every visual)

---

## Relationships

### Incoming Relationships (Source Relationship)

**From Dim_WorkTemplates** (Calculated Table Source)
- **Type:** Calculated table derivation (not a formal relationship)
- **Logic:** Filtered and summarized subset of Dim_WorkTemplates
- **Refresh:** Updates when Dim_WorkTemplates refreshes

### Outgoing Relationships

**To Template_Group_WorkTemplates** (One-to-Many)
- **Relationship ID:** 292b26b0-33cf-5f9e-0ae0-2115cd7ac91a (from source Dim_WorkTemplates)
- **Relationship ID:** 302 (from calculated table)
- **From Column:** Dim_Published_Worktemplates[Template_Link]
- **To Column:** Template_Group_WorkTemplates[Template_Link]
- **Purpose:** Links published templates to template groups via bridge table. Enables many-to-many grouping of templates.

**To Fact_Fragment_Details** (One-to-Many)
- **Relationship ID:** 9fd66eeb-df56-141e-2f99-2872025ee5cc
- **From Column:** Dim_Published_Worktemplates[Id]
- **To Column:** Fact_Fragment_Details[WorkTemplate_Id]
- **Purpose:** Links published templates to detailed fragment specifications in fact table. Enables drill-down from template to its groups, sections, and fields.

**To Dim_Fragment_Type** (Many-to-One, Inactive)
- **Relationship ID:** 0c527108-8ed7-5582-fd03-7d6724739516
- **From Column:** Dim_Published_Worktemplates[Fragment_Type]
- **To Column:** Dim_Fragment_Type[Fragment_Type]
- **Active:** False
- **Purpose:** Inactive role-playing relationship for fragment type classification. Activated via USERELATIONSHIP in DAX measures when fragment type filtering is needed.

---

## Key Usage Patterns

### 1. Published Template Filtering

**Pattern:** Use this table instead of Dim_WorkTemplates for operational reports

**Example: Template Dropdown Slicer**
- Use Dim_Published_Worktemplates[Name] as slicer
- Automatically excludes draft and unpublished templates
- Cleaner user experience

**Example: Published Template Count**
```dax
Published Template Count = 
COUNTROWS(Dim_Published_Worktemplates)
```

---

### 2. Template URL Deep Linking

**Pattern:** Use Template_URL for navigation to template designer

**Example: Template Designer Link in Table Visual**
- Column: Dim_Published_Worktemplates[Name]
- Column: Dim_Published_Worktemplates[Template_URL]
- Format Template_URL column as "Web URL" type
- Users can click to open template in designer

**Example: Dynamic Template Link Measure**
```dax
Template Designer Link = 
VAR SelectedTemplate = SELECTEDVALUE(Dim_Published_Worktemplates[Name])
VAR TemplateURL = SELECTEDVALUE(Dim_Published_Worktemplates[Template_URL])
RETURN
IF(
    NOT ISBLANK(SelectedTemplate),
    TemplateURL,
    BLANK()
)
```

---

### 3. Template Group Analysis

**Pattern:** Analyze template groupings and categorization

**Example: Templates by Group**
```dax
EVALUATE
SUMMARIZE(
    Template_Group_WorkTemplates,
    RELATED(Dim_Template_Groups[Template_Group_Name]),
    RELATED(Dim_Published_Worktemplates[Name])
)
ORDER BY [Template_Group_Name], [Name]
```

**Example: Templates in Multiple Groups**
```dax
Templates in Multiple Groups = 
CALCULATE(
    COUNTROWS(Dim_Published_Worktemplates),
    FILTER(
        Dim_Published_Worktemplates,
        CALCULATE(COUNTROWS(Template_Group_WorkTemplates)) > 1
    )
)
```

---

### 4. Fragment Type Classification (Using Inactive Relationship)

**Pattern:** Use USERELATIONSHIP to filter by fragment type

**Example: Published Work Templates Only**
```dax
Published Work Templates Count = 
CALCULATE(
    COUNTROWS(Dim_Published_Worktemplates),
    USERELATIONSHIP(
        Dim_Published_Worktemplates[Fragment_Type], 
        Dim_Fragment_Type[Fragment_Type]
    ),
    Dim_Fragment_Type[Fragment_Type_Name] = "Work Template"
)
```

---

## Common DAX Patterns

### List All Published Templates
```dax
Published Template List = 
CONCATENATEX(
    Dim_Published_Worktemplates,
    Dim_Published_Worktemplates[Name],
    ", ",
    Dim_Published_Worktemplates[Name],
    ASC
)
```

### Template Published Date Range
```dax
Template Publication Date Range = 
FORMAT(MIN(Dim_Published_Worktemplates[Published_At]), "dd/MM/yyyy") & 
" to " & 
FORMAT(MAX(Dim_Published_Worktemplates[Published_At]), "dd/MM/yyyy")
```

### Count Assignments by Published Template
```dax
Assignments per Template = 
CALCULATE(
    COUNTROWS(Assignments),
    RELATEDTABLE(Fact_Fragment_Details)
)
```

### Recently Published Templates
```dax
Recently Published Templates = 
CALCULATE(
    COUNTROWS(Dim_Published_Worktemplates),
    Dim_Published_Worktemplates[Published_At] >= TODAY() - 30
)
```

### Template Fragment Detail Count
```dax
Fragment Detail Count = 
CALCULATE(
    COUNTROWS(Fact_Fragment_Details),
    USERELATIONSHIP(
        Dim_Published_Worktemplates[Id],
        Fact_Fragment_Details[WorkTemplate_Id]
    )
)
```

---

## Comparison with Dim_WorkTemplates

| Aspect | Dim_Published_Worktemplates | Dim_WorkTemplates |
|--------|----------------------------|-------------------|
| **Scope** | Published templates only | All templates (draft, published, archived) |
| **Table Type** | Calculated table | Source table from database |
| **Columns** | 7 columns (subset) | 12+ columns (full set) |
| **Is_Published Filter** | Pre-applied (all rows TRUE) | Must filter manually |
| **Use Case** | Operational reports, slicers | Template management, admin reports |
| **Refresh** | Automatic (when source refreshes) | From database query |
| **Relationships** | 3 relationships (2 active, 1 inactive) | Multiple relationships to various tables |

---

## Data Quality Considerations

### Referential Integrity
- **Template_Link:** Must exist in source Dim_WorkTemplates
- **Id:** Must be unique across all published templates
- **Fragment_Type:** Should exist in Dim_Fragment_Type (inactive relationship)

### Data Validation
- **Published_At:** Should not be BLANK for published templates
- **Template_URL:** Should be well-formed URL when calculated
- **Name:** Should be unique and descriptive

### Troubleshooting

**Issue: Template appears in Dim_WorkTemplates but not here**
- **Cause:** Is_Published = FALSE or NULL in source table
- **Solution:** Update Is_Published flag in Dim_WorkTemplates

**Issue: Template_URL showing BLANK**
- **Cause:** Dim_Tenant[Tenant_URL] is BLANK or multi-tenant context
- **Solution:** Ensure single tenant selection; verify Tenant_URL populated

**Issue: Relationship to Fact_Fragment_Details not working**
- **Cause:** Id column mismatch or orphaned WorkTemplate_Id in fact table
- **Solution:** Validate referential integrity; check for data quality issues

---

## Performance Considerations

### Calculated Table Overhead
- **Storage:** Minimal (virtual table, computed at query time)
- **Refresh Time:** Negligible (filtering is fast)
- **Query Performance:** Comparable to source table (indexed by Power BI engine)

### Optimization Tips
1. Use this table for slicers instead of filtering Dim_WorkTemplates
2. Leverage pre-filtered state to simplify DAX measures
3. Avoid redundant Is_Published filters in calculations
4. Consider materialized table if source table is very large (>100K rows)

---

## Related Tables

### Source Table
- **Dim_WorkTemplates** - Full work template catalog (see ERD #1)

### Bridge Tables
- **Template_Group_WorkTemplates** - Many-to-many template group membership

### Fact Tables
- **Fact_Fragment_Details** - Detailed fragment specifications by template

### Lookup Tables
- **Dim_Fragment_Type** - Fragment type classification (inactive relationship)

### Filtered Dimension Tables (Derived from Dim_Tenant)
- **WorkTemplate_Filtered_1** - Tenant-specific template list 1
- **WorkTemplate_Filtered_2** - Tenant-specific template list 2

---

## Data Model Position

**Related ERDs:**
- **ERD #6: Templates, Fragments & Configuration** - Primary documentation
- **ERD #1: Assignment Core Model** - Dim_WorkTemplates source table
- **ERD #3: Field Measurements & Time Series** - Templates define measurement structures

**Model Layer:** Dimension (Calculated)

**Refresh Dependency:** Refreshes automatically when Dim_WorkTemplates refreshes

**Lineage Tag:** 212311db-b240-45df-ab1a-3f25220ef377 (or similar, from TMDL)

---

## Change History

| Date | Change Description | Modified By |
|------|-------------------|-------------|
| 2025-11-19 | Initial table documentation created | AI Documentation Generator |

---

## Additional Notes

### Calculated Table Pattern Benefits
This table demonstrates the calculated table pattern for derived dimensions:
- **Simplified Reporting:** Pre-filtered for published templates
- **Reduced Complexity:** Fewer columns than source table
- **Dynamic Updates:** Automatically reflects source table changes
- **No Duplication:** Virtual table with minimal storage overhead
- **Maintains Relationships:** Retains relationship definitions from source

### When to Use Calculated Tables
Best suited for:
- Filtering common subsets (published, active, current)
- Simplifying complex dimensions for report consumers
- Creating role-playing dimensions with subsets
- Reducing cognitive load in field lists

Not ideal for:
- Very large tables (>1M rows) - consider views or source filtering
- Frequent schema changes (requires republishing)
- Complex transformations better handled in Power Query
