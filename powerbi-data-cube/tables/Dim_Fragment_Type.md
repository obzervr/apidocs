# Dim_Fragment_Type

**Table Type:** Lookup Table (Static)

**Purpose:** Static lookup table defining fragment type classifications. Provides standardized fragment type codes and display names with sort ordering.

**Last Updated:** November 19, 2025

---

## Overview

Dim_Fragment_Type is a static reference table that defines the types of fragments used in the Obzervr template system. It serves as a role-playing dimension with multiple inactive relationships to various fragment tables, enabling fragment type filtering via USERELATIONSHIP in DAX measures.

---

## Columns

| Column Name | Data Type | Description | Sort |
|------------|-----------|-------------|------|
| **Fragment_Type** | String | Fragment type code (PK) | Sorted by Sort_Order |
| **Fragment_Type_Name** | String | Fragment type display name | Sorted by Sort_Order |
| **Sort_Order** | Integer | Sort order for display | - |

---

## Fragment Types

Expected values (from compressed JSON source):

| Fragment_Type | Fragment_Type_Name | Sort_Order |
|--------------|-------------------|------------|
| WorkTemplate | Work Template | 0 |
| GroupFragment | Group Fragment | 1 |
| SectionFragment | Section Fragment | 2 |
| FieldFragment | Field Fragment | 3 |

---

## Relationships (All Inactive)

**From Dim_Published_Worktemplates** (Inactive)
- **Relationship ID:** 0c527108-8ed7-5582-fd03-7d6724739516
- **From:** Dim_Published_Worktemplates[Fragment_Type]
- **To:** Dim_Fragment_Type[Fragment_Type]
- **Purpose:** Fragment type classification for templates

**From Dim_Published_Group_Fragments** (Inactive)
- **Relationship ID:** AutoDetected_862df2ed-8596-413b-b180-585fde0873e2
- **From:** Dim_Published_Group_Fragments[Fragment_Type]
- **To:** Dim_Fragment_Type[Fragment_Type]
- **Purpose:** Fragment type classification for groups

**From Dim_Published_Section_Fragments** (Inactive)
- **Relationship ID:** AutoDetected_245dab17-f66d-4ec8-b4ff-193ea19c0db9
- **From:** Dim_Published_Section_Fragments[Fragment_Type]
- **To:** Dim_Fragment_Type[Fragment_Type]
- **Purpose:** Fragment type classification for sections

**From Dim_Published_Field_Fragments** (Inactive)
- **Relationship ID:** AutoDetected_af84b4f1-17bd-489f-8556-719a0f255a0b
- **From:** Dim_Published_Field_Fragments[Fragment_Type]
- **To:** Dim_Fragment_Type[Fragment_Type]
- **Purpose:** Fragment type classification for fields

---

## Data Source

**Source Type:** Power Query M (Table.FromRows)

```m
let
    Source = Table.FromRows(
        Json.Document(
            Binary.Decompress(
                Binary.FromText("...", BinaryEncoding.Base64), 
                Compression.Deflate
            )
        ), 
        ...
    ),
    #"Changed Type" = Table.TransformColumnTypes(
        Source,
        {
            {"Fragment_Type", type text}, 
            {"Fragment_Type_Name", type text},
            {"Sort_Order", Int64.Type}
        }
    )
in
    #"Changed Type"
```

**Pattern:** Static embedded JSON data with binary compression

---

## Usage with USERELATIONSHIP

### Why All Relationships Are Inactive

**Design Rationale:**
- Each fragment table (templates, groups, sections, fields) has a Fragment_Type column
- Only one active relationship per column is allowed in Power BI
- Making all relationships inactive avoids ambiguity
- Explicit USERELATIONSHIP calls make DAX code clearer

### Common DAX Patterns

**Count Group Fragments**
```dax
Group Fragment Count = 
CALCULATE(
    COUNTROWS(Dim_Published_Group_Fragments),
    USERELATIONSHIP(
        Dim_Published_Group_Fragments[Fragment_Type], 
        Dim_Fragment_Type[Fragment_Type]
    ),
    Dim_Fragment_Type[Fragment_Type_Name] = "Group Fragment"
)
```

**Fragment Count by Type**
```dax
EVALUATE
ADDCOLUMNS(
    Dim_Fragment_Type,
    "Group Count", CALCULATE(
        COUNTROWS(Dim_Published_Group_Fragments),
        USERELATIONSHIP(
            Dim_Published_Group_Fragments[Fragment_Type], 
            Dim_Fragment_Type[Fragment_Type]
        )
    ),
    "Section Count", CALCULATE(
        COUNTROWS(Dim_Published_Section_Fragments),
        USERELATIONSHIP(
            Dim_Published_Section_Fragments[Fragment_Type], 
            Dim_Fragment_Type[Fragment_Type]
        )
    ),
    "Field Count", CALCULATE(
        COUNTROWS(Dim_Published_Field_Fragments),
        USERELATIONSHIP(
            Dim_Published_Field_Fragments[Fragment_Type], 
            Dim_Fragment_Type[Fragment_Type]
        )
    )
)
```

**Filter Fragments by Type Name**
```dax
Section Fragments Only = 
FILTER(
    Dim_Fragment_Type,
    Dim_Fragment_Type[Fragment_Type_Name] = "Section Fragment"
)
```

---

## Related Tables

### Fragment Tables (All with Inactive Relationships)
- **Dim_Published_Worktemplates** - Work templates
- **Dim_Published_Group_Fragments** - Group fragments
- **Dim_Published_Section_Fragments** - Section fragments
- **Dim_Published_Field_Fragments** - Field fragments

---

## Data Model Position

**Related ERDs:** ERD #6 (Templates, Fragments & Configuration)  
**Model Layer:** Lookup Table (Static Reference)

---

## Change History

| Date | Change Description | Modified By |
|------|-------------------|-------------|
| 2025-11-19 | Initial table documentation created | AI Documentation Generator |

---

## Additional Notes

### Role-Playing Dimension Pattern
This table demonstrates the role-playing dimension pattern with all-inactive relationships:
- **Single Source of Truth:** One table defines all fragment types
- **Multiple Contexts:** Same types used across different fragment tables
- **Explicit Activation:** USERELATIONSHIP makes context clear in DAX
- **Avoids Ambiguity:** No confusion about which relationship is active

### Static Reference Table Benefits
- **No Refresh Needed:** Data is embedded in model (binary compressed JSON)
- **Performance:** Extremely fast (in-memory, no database query)
- **Consistency:** Fragment types cannot drift between environments
- **Version Control:** Changes require model republishing (intentional governance)
