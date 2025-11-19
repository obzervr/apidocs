# Dim_Published_Section_Fragments

**Table Type:** Dimension Table

**Purpose:** Published section fragment definitions. Sections represent the middle tier of the fragment hierarchy within groups (Group → **Section** → Field).

**Last Updated:** November 19, 2025

---

## Overview

Dim_Published_Section_Fragments contains published section definitions that organize fields within group fragments. Sections provide logical grouping of related data capture fields, creating a three-tier template hierarchy that structures data collection in Obzervr work templates.

**Hierarchy Position:** Level 2 of 3 (Group → **Section** → Field)

---

## Columns

| Column Name | Data Type | Description | Key |
|------------|-----------|-------------|-----|
| **Section_Id** | String | Section fragment identifier | PK |
| **Tenant_Id** | String | Tenant identifier for multi-tenancy | - |
| **Template_Link** | String | Reference to parent template | FK |
| **Identifier** | String | Internal section identifier | - |
| **Name** | String | Section name | - |
| **Description** | String | Section description text | - |
| **Fragment_Type** | String | Fragment type (typically "SectionFragment") | FK |
| *(Additional columns similar to group fragments)* | | Including Published_At, Version, Created_Date, etc. | - |

---

## Relationships

**To Fact_Fragment_Details** (One-to-Many)
- **Relationship ID:** 0485ace5-b38f-3fe8-3457-eddf81222168
- **From:** Dim_Published_Section_Fragments[Section_Id]
- **To:** Fact_Fragment_Details[Section_Id]
- **Purpose:** Links sections to detailed field specifications

**To Dim_Fragment_Type** (Many-to-One, Inactive)
- **Relationship ID:** AutoDetected_245dab17-f66d-4ec8-b4ff-193ea19c0db9
- **From:** Dim_Published_Section_Fragments[Fragment_Type]
- **To:** Dim_Fragment_Type[Fragment_Type]
- **Purpose:** Fragment type classification (use with USERELATIONSHIP)

---

## Data Source

**Source Type:** SQL Query  
**Source Table/View:** VW_TemplateSectionFragments (inferred)

Filtered by tenant (AllTenants or TenantId1-5 parameters).

---

## Example: Section in Template Hierarchy

**Template:** "Equipment Pre-Start Inspection"
- **Group:** "Engine Checks"
  - **Section:** "Oil System" ← This table
    - Field: Oil Level
    - Field: Oil Condition
    - Field: Oil Leak Check
  - **Section:** "Cooling System" ← This table
    - Field: Coolant Level
    - Field: Hose Condition
    - Field: Radiator Inspection

Sections organize related fields within a group, improving data capture UX and reporting structure.

---

## Common DAX Patterns

### Count Sections per Group
```dax
Sections per Group = 
CALCULATE(
    DISTINCTCOUNT(Dim_Published_Section_Fragments[Section_Id]),
    ALLEXCEPT(
        Fact_Fragment_Details, 
        Fact_Fragment_Details[Group_Id]
    )
)
```

### Section Count by Template
```dax
EVALUATE
SUMMARIZE(
    Dim_Published_Section_Fragments,
    Dim_Published_Section_Fragments[Template_Link],
    "Section Count", COUNTROWS(Dim_Published_Section_Fragments)
)
```

---

## Related Tables

### Fragment Hierarchy
- **Dim_Published_Group_Fragments** - Parent groups (level above)
- **Dim_Published_Field_Fragments** - Child fields (level below)
- **Dim_Published_Worktemplates** - Top-level templates
- **Fact_Fragment_Details** - Detailed specifications (snowflake fact)

### Lookup
- **Dim_Fragment_Type** - Fragment type classification

---

## Data Model Position

**Related ERDs:** ERD #6 (Templates, Fragments & Configuration)  
**Model Layer:** Dimension (Snowflake Schema - Level 2)

---

## Change History

| Date | Change Description | Modified By |
|------|-------------------|-------------|
| 2025-11-19 | Initial table documentation created | AI Documentation Generator |
