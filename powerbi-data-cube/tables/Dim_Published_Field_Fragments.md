# Dim_Published_Field_Fragments

**Table Type:** Dimension Table

**Purpose:** Published field fragment definitions. Fields represent the lowest tier of the fragment hierarchy, containing actual data capture specifications (Group → Section → **Field**).

**Last Updated:** November 19, 2025

---

## Overview

Dim_Published_Field_Fragments contains published field definitions that specify individual data capture points within sections. Fields are the atomic units of data collection, defining the type of data captured (numeric, text, photo, dropdown, etc.), validation rules, and display properties.

**Hierarchy Position:** Level 3 of 3 (Group → Section → **Field**)

---

## Columns

| Column Name | Data Type | Description | Key |
|------------|-----------|-------------|-----|
| **Field_Id** | String | Field fragment identifier | PK |
| **Tenant_Id** | String | Tenant identifier for multi-tenancy | - |
| **Template_Link** | String | Reference to parent template | FK |
| **Identifier** | String | Internal field identifier | - |
| **Name** | String | Field name | - |
| **Description** | String | Field description text | - |
| **Fragment_Type** | String | Fragment type (typically "FieldFragment") | FK |
| *(Additional columns)* | | Including field type, validation rules, etc. | - |

---

## Relationships

**To Fact_Fragment_Details** (One-to-Many)
- **Relationship ID:** 550ce7e9-3724-1202-77fb-33d30235e527
- **From:** Dim_Published_Field_Fragments[Field_Id]
- **To:** Fact_Fragment_Details[Field_Id]
- **Purpose:** Links fields to detailed specifications including validation boundaries, help text, and field types

**To Dim_Fragment_Type** (Many-to-One, Inactive)
- **Relationship ID:** AutoDetected_af84b4f1-17bd-489f-8556-719a0f255a0b
- **From:** Dim_Published_Field_Fragments[Fragment_Type]
- **To:** Dim_Fragment_Type[Fragment_Type]
- **Purpose:** Fragment type classification (use with USERELATIONSHIP)

---

## Data Source

**Source Type:** SQL Query  
**Source Table/View:** VW_TemplateFieldFragments (inferred)

Filtered by tenant (AllTenants or TenantId1-5 parameters).

---

## Field Types

Fields in Fact_Fragment_Details define the data capture control type:
- **Numeric:** Number input with validation boundaries
- **Text:** Free text input
- **Dropdown:** Selection from predefined list
- **Photo:** Image capture
- **Date/Time:** Date or datetime picker
- **Boolean:** Yes/No or True/False
- **Signature:** Digital signature capture
- **Barcode/QR:** Scanner input

---

## Example: Fields in Template Hierarchy

**Template:** "Equipment Pre-Start Inspection"
- **Group:** "Engine Checks"
  - **Section:** "Oil System"
    - **Field:** "Oil Level (Liters)" ← This table (Numeric, Range: 0-10)
    - **Field:** "Oil Condition" ← This table (Dropdown: Good/Fair/Poor)
    - **Field:** "Oil Leak Photo" ← This table (Photo)

Fields are the actual data entry points that operators interact with in the Obzervr mobile/web app.

---

## Common DAX Patterns

### Count Fields per Section
```dax
Fields per Section = 
CALCULATE(
    DISTINCTCOUNT(Dim_Published_Field_Fragments[Field_Id]),
    ALLEXCEPT(
        Fact_Fragment_Details, 
        Fact_Fragment_Details[Section_Id]
    )
)
```

### Field Count by Type
```dax
EVALUATE
SUMMARIZE(
    Fact_Fragment_Details,
    Fact_Fragment_Details[Field_Type],
    "Field Count", COUNTROWS(Fact_Fragment_Details)
)
ORDER BY [Field Count] DESC
```

### Required Fields
```dax
Required Field Count = 
CALCULATE(
    COUNTROWS(Fact_Fragment_Details),
    Fact_Fragment_Details[Is_Required] = "true"
)
```

---

## Related Tables

### Fragment Hierarchy
- **Dim_Published_Group_Fragments** - Parent groups (2 levels up)
- **Dim_Published_Section_Fragments** - Parent sections (1 level up)
- **Dim_Published_Worktemplates** - Top-level templates
- **Fact_Fragment_Details** - Detailed field specifications with validation rules

### Operational Data
- **TimeSeries_FieldMeasurements** - Actual captured field data in assignments

### Lookup
- **Dim_Fragment_Type** - Fragment type classification

---

## Data Model Position

**Related ERDs:** ERD #6 (Templates, Fragments & Configuration)  
**Model Layer:** Dimension (Snowflake Schema - Level 3, Lowest)

---

## Change History

| Date | Change Description | Modified By |
|------|-------------------|-------------|
| 2025-11-19 | Initial table documentation created | AI Documentation Generator |

---

## Additional Notes

### Field to FieldMeasurement Mapping
When templates are used in assignments:
- **Dim_Published_Field_Fragments** defines the template structure
- **TimeSeries_FieldMeasurements** contains actual captured values
- **FieldMeasurement_Identifier** in TimeSeries_FieldMeasurements links to Field Identifier

This separation enables template evolution without breaking historical data.
