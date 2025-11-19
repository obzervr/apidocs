# Dim_Fragments

## Table Overview
`Dim_Fragments` is a dimension table that catalogues template fragments - the building blocks that compose work templates. Fragments represent groups, sections, and individual fields that can be assembled into complete data collection forms. Each fragment has a type classification and version control.

This table shares the same source as `Dim_WorkTemplates` but filters to fragment types (GroupFragment, FieldFragment, SectionFragment) rather than work templates.

**Current Status**: Standard dimension table sourced from same table as work templates.

---

## Specifications
- **Source**: `DimWorkTemplates` table (same source as `Dim_WorkTemplates`)
- **Row Count**: Moderate to high volume (hundreds of fragments per tenant)
- **Grain**: One row per fragment version
- **Primary Key**: `Id`
- **Incremental Refresh**: Not enabled
- **Partitioning Strategy**: Standard import
- **Source Columns**: 12
- **Calculated Columns**: 0
- **Filtering**: FragmentType IN ('GroupFragment', 'FieldFragment', 'SectionFragment')

---

## Column Specifications

| Column Name | Data Type | Format | Nullable | Hidden | Description |
|------------|-----------|--------|----------|--------|-------------|
| Id | Int64 | | No | No | Primary key for fragment |
| Tenant_Id | Int64 | | No | No | Foreign key to tenant dimension |
| Identifier | String | | No | No | Unique identifier code for the fragment |
| Name | String | | No | No | Display name of the fragment |
| Version | String | | Yes | No | Fragment version number (e.g., "1.0", "2.1") |
| Is_Published | Boolean | true/false | No | No | Indicates if fragment is published and available for use |
| Fragment_Type | String | | No | No | Type classification: "GroupFragment", "FieldFragment", or "SectionFragment" |
| Created_Date | DateTime | yyyy-MM-dd HH:mm:ss | No | No | Timestamp when fragment was created |
| Last_Updated | DateTime | yyyy-MM-dd HH:mm:ss | Yes | No | Timestamp of last modification |
| TemplateLink | String | | Yes | No | URL or identifier linking to fragment definition |
| Json | String | | Yes | No | JSON representation of fragment structure |
| LastLoaded | DateTime | yyyy-MM-dd HH:mm:ss | Yes | No | ETL timestamp for data lineage |

---

## Calculated Columns
None. This table uses only source columns from the dimension table.

---

## Relationships

### Outbound Relationships
| To Table | From Column(s) | To Column(s) | Cardinality | Cross Filter | Relationship ID |
|----------|---------------|--------------|-------------|--------------|-----------------|
| `Dim_Tenant` | Tenant_Id | Tenant_Id | Many-to-One | Single | (Tenant context) |

### Inbound Relationships
| From Table | From Column(s) | To Column(s) | Cardinality | Cross Filter | Relationship ID |
|-----------|---------------|--------------|-------------|--------------|-----------------|
| `Fact_Fragment_Details` | Group_Id | Id | Many-to-One | Single | (Group fragment context) |
| `Fact_Fragment_Details` | Section_Id | Id | Many-to-One | Single | (Section fragment context) |
| `Fact_Fragment_Details` | Field_Id | Id | Many-to-One | Single | (Field fragment context) |

---

## Power Query M Source

```m
let
    Source = Value.NativeQuery(
        Obzervr_DataWarehouse,
        "SELECT 
            Id,
            Tenant_Id,
            Identifier,
            Name,
            Version,
            Is_Published,
            Fragment_Type,
            Created_Date,
            Last_Updated,
            TemplateLink,
            Json,
            LastLoaded
        FROM [dbo].[DimWorkTemplates]
        WHERE Fragment_Type IN ('GroupFragment', 'FieldFragment', 'SectionFragment')"
    ),
    #"Filtered Tenant" = Table.SelectRows(
        Source,
        each [Tenant_Id] = TenantId
    ),
    #"Changed Types" = Table.TransformColumnTypes(
        #"Filtered Tenant",
        {
            {"Id", Int64.Type},
            {"Tenant_Id", Int64.Type},
            {"Identifier", type text},
            {"Name", type text},
            {"Version", type text},
            {"Is_Published", type logical},
            {"Fragment_Type", type text},
            {"Created_Date", type datetime},
            {"Last_Updated", type datetime},
            {"TemplateLink", type text},
            {"Json", type text},
            {"LastLoaded", type datetime}
        }
    )
in
    #"Changed Types"
```

---

## DAX Query Patterns

### Example 1: Fragment Distribution by Type
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_Fragments[Fragment_Type],
    "Fragment_Count", COUNTROWS(Dim_Fragments),
    "Published_Count", COUNTROWS(
        FILTER(Dim_Fragments, Dim_Fragments[Is_Published] = TRUE())
    ),
    "Unique_Identifiers", DISTINCTCOUNT(Dim_Fragments[Identifier]),
    "Latest_Update", MAX(Dim_Fragments[Last_Updated])
)
ORDER BY Dim_Fragments[Fragment_Type]
```

### Example 2: Published Field Fragments
```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Dim_Fragments,
        "Fragment_Id", Dim_Fragments[Id],
        "Fragment_Name", Dim_Fragments[Name],
        "Identifier", Dim_Fragments[Identifier],
        "Type", Dim_Fragments[Fragment_Type],
        "Version", Dim_Fragments[Version],
        "Created", FORMAT(Dim_Fragments[Created_Date], "yyyy-MM-dd")
    ),
    Dim_Fragments[Fragment_Type] = "FieldFragment"
    && Dim_Fragments[Is_Published] = TRUE()
)
ORDER BY Dim_Fragments[Name]
```

### Example 3: Fragment Version Analysis
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_Fragments[Name],
    Dim_Fragments[Fragment_Type],
    "Version_Count", COUNTROWS(Dim_Fragments),
    "Latest_Version", MAX(Dim_Fragments[Version]),
    "Published_Versions", COUNTROWS(
        FILTER(Dim_Fragments, Dim_Fragments[Is_Published] = TRUE())
    ),
    "Latest_Update", MAX(Dim_Fragments[Last_Updated])
)
ORDER BY [Version_Count] DESC
```

### Example 4: Fragment Usage in Templates
```dax
EVALUATE
ADDCOLUMNS(
    FILTER(
        Dim_Fragments,
        Dim_Fragments[Fragment_Type] = "GroupFragment"
    ),
    "Used_In_Templates", CALCULATE(
        DISTINCTCOUNT(Fact_Fragment_Details[WorkTemplate_Id])
    ),
    "Section_Count", CALCULATE(
        DISTINCTCOUNT(Fact_Fragment_Details[Section_Id]),
        USERELATIONSHIP(Fact_Fragment_Details[Group_Id], Dim_Fragments[Id])
    ),
    "Field_Count", CALCULATE(
        DISTINCTCOUNT(Fact_Fragment_Details[Field_Id]),
        USERELATIONSHIP(Fact_Fragment_Details[Group_Id], Dim_Fragments[Id])
    )
)
ORDER BY [Used_In_Templates] DESC
```

### Example 5: Recent Fragment Updates
```dax
EVALUATE
TOPN(
    50,
    SELECTCOLUMNS(
        Dim_Fragments,
        "Fragment_Name", Dim_Fragments[Name],
        "Type", Dim_Fragments[Fragment_Type],
        "Version", Dim_Fragments[Version],
        "Published", Dim_Fragments[Is_Published],
        "Last_Updated", FORMAT(Dim_Fragments[Last_Updated], "yyyy-MM-dd HH:mm"),
        "Days_Since_Update", DATEDIFF(Dim_Fragments[Last_Updated], TODAY(), DAY)
    ),
    Dim_Fragments[Last_Updated],
    DESC
)
```

### Example 6: Fragments Without Template Links
```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Dim_Fragments,
        "Fragment_Id", Dim_Fragments[Id],
        "Fragment_Name", Dim_Fragments[Name],
        "Type", Dim_Fragments[Fragment_Type],
        "Identifier", Dim_Fragments[Identifier],
        "Published", Dim_Fragments[Is_Published],
        "TemplateLink", Dim_Fragments[TemplateLink]
    ),
    ISBLANK(Dim_Fragments[TemplateLink])
)
ORDER BY Dim_Fragments[Fragment_Type], Dim_Fragments[Name]
```

---

## Data Model Pattern

### Fragment Catalog Pattern
`Dim_Fragments` implements a fragment catalog pattern where reusable template components are stored as independent entities that can be composed into complete work templates. This enables modular template design and fragment reuse across multiple templates.

**Fragment Type Hierarchy**:
Fragments exist in a 3-level hierarchy within templates:

1. **GroupFragment**: Top-level organizational container
   - Groups major sections of related content
   - Example: "Safety Checks", "Equipment Inspection", "Sign-Off"
   - Contains multiple sections

2. **SectionFragment**: Mid-level organizational container
   - Subsections within groups
   - Example: "Personal Protective Equipment", "Engine Compartment", "Tires"
   - Contains multiple fields

3. **FieldFragment**: Leaf-level data collection field
   - Individual data entry fields
   - Example: "Engine Oil Level", "Hard Hat Worn", "Inspector Signature"
   - Defines field type, boundaries, UOM, validation rules

**Fragment Composition Pattern**:
```
Work Template (Dim_WorkTemplates)
├── Group Fragment 1 (Dim_Fragments)
│   ├── Section Fragment 1a (Dim_Fragments)
│   │   ├── Field Fragment 1a-i (Dim_Fragments)
│   │   ├── Field Fragment 1a-ii (Dim_Fragments)
│   │   └── Field Fragment 1a-iii (Dim_Fragments)
│   └── Section Fragment 1b (Dim_Fragments)
│       ├── Field Fragment 1b-i (Dim_Fragments)
│       └── Field Fragment 1b-ii (Dim_Fragments)
└── Group Fragment 2 (Dim_Fragments)
    └── Section Fragment 2a (Dim_Fragments)
        └── Field Fragment 2a-i (Dim_Fragments)
```

The actual composition is stored in `Fact_Fragment_Details`, which links fragments to work templates with sequence numbering and additional context.

**Fragment Reusability**:
Fragments are designed for reuse:
- A single "Engine Oil Level" field fragment can be used in multiple templates
- Common section fragments (e.g., "Safety Checks") can be reused across work types
- Group fragments provide consistent structure across related templates
- Fragment versioning enables evolution without breaking existing templates

**Version Control**:
Fragment versioning works similarly to template versioning:
- Multiple versions of the same fragment can coexist (same `Identifier`, different `Version`)
- `Is_Published` determines which versions are available for template composition
- Historical templates retain links to the fragment versions used at creation time
- Version evolution enables gradual migration (e.g., updating field boundaries)

**Shared Source Table Pattern**:
`Dim_Fragments` and `Dim_WorkTemplates` source from the same physical table (`DimWorkTemplates`) but filter on different `Fragment_Type` values:
- **Dim_WorkTemplates**: `Fragment_Type = 'WorkTemplate'`
- **Dim_Fragments**: `Fragment_Type IN ('GroupFragment', 'FieldFragment', 'SectionFragment')`

This pattern enables:
- Unified storage of template components
- Consistent versioning and publishing workflow
- Single source for template metadata
- Simplified template management

**TemplateLink vs Template_Link**:
Note the column name difference between tables:
- **Dim_WorkTemplates**: Uses `Template_Link` (underscore) and requires NOT NULL
- **Dim_Fragments**: Uses `TemplateLink` (no underscore) and allows NULL

This suggests:
- Fragments may not always have direct edit links (reusable library components)
- Work templates must have edit links (active forms requiring management)

**JSON Structure**:
The `Json` column stores fragment definitions including:
- For **FieldFragment**: Field type, validation rules, boundaries, UOM, help text
- For **SectionFragment**: Section layout, field ordering
- For **GroupFragment**: Group organization, section ordering

**Example Scenario - Field Fragment Reuse**:

**Field Fragment**: "Engine Oil Level" (FieldFragment)
- **Id**: 5001
- **Identifier**: "FLD-OIL-LEVEL"
- **Version**: "1.0"
- **Fragment_Type**: "FieldFragment"
- **Is_Published**: TRUE
- **Json**: `{"fieldType": "Number", "UOM": "Liters", "lowerBoundary": 7, "upperBoundary": 10, "helpText": "Check oil level with dipstick"}`

**Used In Multiple Templates**:
1. "Haul Truck Pre-Start Inspection" (WorkTemplate Id 101)
2. "Excavator Daily Check" (WorkTemplate Id 102)
3. "Light Vehicle Inspection" (WorkTemplate Id 103)
4. "Generator Pre-Start Check" (WorkTemplate Id 104)

By storing this field fragment once and reusing it across templates:
- Updates to boundaries (e.g., changing range to 6-11 liters) can be made in one place
- Consistency across templates is maintained
- Reporting can aggregate "Engine Oil Level" readings across all equipment types
- Template creation is faster (select from library rather than recreate fields)

**Fragment Lifecycle**:
1. **Creation**: Fragment created with `Is_Published = FALSE` (draft state)
2. **Testing**: Fragment tested in draft templates
3. **Publishing**: Fragment published (`Is_Published = TRUE`)
4. **Usage**: Fragment added to work templates via `Fact_Fragment_Details`
5. **Evolution**: New version created (Version 2.0), old version retained for historical templates
6. **Retirement**: Old version set to `Is_Published = FALSE`, no longer available for new templates

---

## Related Documentation
- **ERD_06_Templates_Fragments.md** - ERD diagram showing fragment and template relationship context
- **Dim_WorkTemplates.md** - Work template catalog (top-level templates)
- **Fact_Fragment_Details.md** - Detailed template structure linking fragments to templates
- **TimeSeries_FieldMeasurements.md** - Captured field measurement data using these fragment definitions

---

## Change History
| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-15 | Auto-generated | Initial documentation from TMDL metadata |

---

## Notes
- **Shared Source Table**: This table and `Dim_WorkTemplates` source from the same `DimWorkTemplates` table but filter on different `Fragment_Type` values, demonstrating a type-based partitioning pattern.
- **Three Fragment Types**: The WHERE clause filters to three fragment types: GroupFragment, FieldFragment, and SectionFragment, representing the 3-level hierarchy within templates.
- **TemplateLink Nullable**: Unlike `Dim_WorkTemplates` which requires `Template_Link IS NOT NULL`, this table allows NULL values in `TemplateLink`, suggesting fragments may be library components without direct edit URLs.
- **Fragment Reusability**: Fragments are designed to be reused across multiple work templates, enabling modular template design and consistent field definitions.
- **Version Control**: Fragment versioning enables evolution without breaking existing templates that reference older versions.
- **Published Status**: Only fragments with `Is_Published = TRUE` should be available for adding to new templates, though historical templates retain links to their original fragment versions.
- **JSON Storage**: The `Json` column stores complete fragment definitions including field types, validation rules, boundaries, and metadata specific to each fragment type.
- **Identifier Stability**: The `Identifier` column provides a stable reference across fragment versions, while `Id` (primary key) is unique per version.
- **Fragment Hierarchy**: Fragment relationships are materialized in `Fact_Fragment_Details`, not directly in this dimension (e.g., which sections belong to which groups).
- **Tenant Isolation**: Fragments are filtered by Tenant_Id in Power Query, maintaining multi-tenant isolation.
- **FieldFragment Examples**: Common field fragments include signatures, dates, booleans (checkboxes), numbers (with boundaries), text, locations, attachments, and multi-select options.
- **SectionFragment Grouping**: Section fragments group related fields logically (e.g., "Personal Protective Equipment" section containing "Hard Hat", "Safety Boots", "High-Vis Vest" field fragments).
- **GroupFragment Organization**: Group fragments provide major organizational divisions (e.g., "Safety Checks", "Equipment Inspection", "Operational Readings", "Sign-Off").
- **No Calculated Columns**: Unlike `Dim_WorkTemplates` which has filter flag columns, this table doesn't include calculated columns, suggesting fragments don't require the same filtered subset pattern.
- **LastLoaded Timestamp**: Enables ETL monitoring and refresh validation.
- **Fragment Evolution**: Creating new fragment versions (incrementing Version) allows gradual migration without breaking existing templates that reference specific versions.
