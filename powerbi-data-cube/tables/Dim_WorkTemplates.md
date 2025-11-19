# Dim_WorkTemplates

## Table Overview
`Dim_WorkTemplates` is a dimension table that catalogues work templates used in the Obzervr system. Work templates define structured workflows and data collection forms that can be assigned to field assignments. Each template consists of organized fragments (groups, sections, fields) that define what data should be collected and how.

This table includes two calculated filter flag columns that indicate whether templates match specific filtered dimensional subsets.

**Current Status**: Standard dimension table with calculated filter flags.

---

## Specifications
- **Source**: `DimWorkTemplates` table
- **Row Count**: Low to moderate volume (typically 10-100 templates per tenant)
- **Grain**: One row per work template version
- **Primary Key**: `Id`
- **Incremental Refresh**: Not enabled
- **Partitioning Strategy**: Standard import
- **Source Columns**: 12
- **Calculated Columns**: 2 (filter flags)
- **Filtering**: FragmentType = "WorkTemplate" AND TemplateLink <> NULL

---

## Column Specifications

| Column Name | Data Type | Format | Nullable | Hidden | Description |
|------------|-----------|--------|----------|--------|-------------|
| Id | Int64 | | No | No | Primary key for work template |
| Tenant_Id | Int64 | | No | No | Foreign key to tenant dimension |
| Identifier | String | | No | No | Unique identifier code for the template |
| Name | String | | No | No | Display name of the work template |
| Version | String | | Yes | No | Template version number (e.g., "1.0", "2.3") |
| Is_Published | Boolean | true/false | No | No | Indicates if template is published and available for use |
| Fragment_Type | String | | No | No | Type classification (filtered to "WorkTemplate" only) |
| Created_Date | DateTime | yyyy-MM-dd HH:mm:ss | No | No | Timestamp when template was created |
| Last_Updated | DateTime | yyyy-MM-dd HH:mm:ss | Yes | No | Timestamp of last modification |
| Template_Link | String | | No | No | URL or identifier linking to template details (filtered to NOT NULL) |
| Json | String | | Yes | No | JSON representation of template structure |
| LastLoaded | DateTime | yyyy-MM-dd HH:mm:ss | Yes | No | ETL timestamp for data lineage |

---

## Calculated Columns

### Is_WorkTemplate_Filtered1
Binary flag (0/1) indicating whether this work template exists in a filtered subset of the dimension (Filtered1 context).

```dax
Is_WorkTemplate_Filtered1 = 
IF(
    ISBLANK(
        LOOKUPVALUE(
            Dim_WorkTemplates_Filtered1[Identifier],
            Dim_WorkTemplates_Filtered1[Identifier], Dim_WorkTemplates[Identifier]
        )
    ),
    0,
    1
)
```

### Is_WorkTemplate_Filtered2
Binary flag (0/1) indicating whether this work template exists in a second filtered subset of the dimension (Filtered2 context).

```dax
Is_WorkTemplate_Filtered2 = 
IF(
    ISBLANK(
        LOOKUPVALUE(
            Dim_WorkTemplates_Filtered2[Identifier],
            Dim_WorkTemplates_Filtered2[Identifier], Dim_WorkTemplates[Identifier]
        )
    ),
    0,
    1
)
```

**Note**: The referenced tables `Dim_WorkTemplates_Filtered1` and `Dim_WorkTemplates_Filtered2` are not documented in the provided TMDL files, suggesting they may be calculated tables or parameter-driven filtered versions of this dimension.

---

## Relationships

### Outbound Relationships
| To Table | From Column(s) | To Column(s) | Cardinality | Cross Filter | Relationship ID |
|----------|---------------|--------------|-------------|--------------|-----------------|
| `Dim_Tenant` | Tenant_Id | Tenant_Id | Many-to-One | Single | (Tenant context) |

### Inbound Relationships
| From Table | From Column(s) | To Column(s) | Cardinality | Cross Filter | Relationship ID |
|-----------|---------------|--------------|-------------|--------------|-----------------|
| `Fact_Fragment_Details` | WorkTemplate_Id | Id | Many-to-One | Single | (Fragment details) |
| `Assignments` | WorkTemplate_Id | Id | Many-to-One | Single | (Assigned templates - if relationship exists) |

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
            Template_Link,
            Json,
            LastLoaded
        FROM [dbo].[DimWorkTemplates]
        WHERE Fragment_Type = 'WorkTemplate'
          AND Template_Link IS NOT NULL"
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
            {"Template_Link", type text},
            {"Json", type text},
            {"LastLoaded", type datetime}
        }
    )
in
    #"Changed Types"
```

---

## DAX Query Patterns

### Example 1: Published Work Templates
```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Dim_WorkTemplates,
        "Template_Id", Dim_WorkTemplates[Id],
        "Template_Name", Dim_WorkTemplates[Name],
        "Identifier", Dim_WorkTemplates[Identifier],
        "Version", Dim_WorkTemplates[Version],
        "Published", Dim_WorkTemplates[Is_Published],
        "Created", FORMAT(Dim_WorkTemplates[Created_Date], "yyyy-MM-dd"),
        "Template_Link", Dim_WorkTemplates[Template_Link]
    ),
    Dim_WorkTemplates[Is_Published] = TRUE()
)
ORDER BY Dim_WorkTemplates[Name]
```

### Example 2: Work Template Usage Analysis
```dax
EVALUATE
ADDCOLUMNS(
    Dim_WorkTemplates,
    "Assignment_Count", CALCULATE(
        COUNTROWS(Assignments),
        USERELATIONSHIP(Assignments[WorkTemplate_Id], Dim_WorkTemplates[Id])
    ),
    "Fragment_Count", CALCULATE(
        DISTINCTCOUNT(Fact_Fragment_Details[Field_Id])
    ),
    "Last_Used", CALCULATE(
        MAX(Assignments[Created_Date]),
        USERELATIONSHIP(Assignments[WorkTemplate_Id], Dim_WorkTemplates[Id])
    )
)
ORDER BY [Assignment_Count] DESC
```

### Example 3: Template Versions
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_WorkTemplates[Name],
    Dim_WorkTemplates[Version],
    Dim_WorkTemplates[Is_Published],
    "Template_Count", COUNTROWS(Dim_WorkTemplates),
    "Latest_Update", MAX(Dim_WorkTemplates[Last_Updated])
)
ORDER BY Dim_WorkTemplates[Name], Dim_WorkTemplates[Version] DESC
```

### Example 4: Filtered Work Template Analysis
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_WorkTemplates[Name],
    "In_Filter1", MAX(Dim_WorkTemplates[Is_WorkTemplate_Filtered1]),
    "In_Filter2", MAX(Dim_WorkTemplates[Is_WorkTemplate_Filtered2]),
    "Filter_Status", 
        SWITCH(
            TRUE(),
            MAX(Dim_WorkTemplates[Is_WorkTemplate_Filtered1]) = 1 
                && MAX(Dim_WorkTemplates[Is_WorkTemplate_Filtered2]) = 1, "Both Filters",
            MAX(Dim_WorkTemplates[Is_WorkTemplate_Filtered1]) = 1, "Filter1 Only",
            MAX(Dim_WorkTemplates[Is_WorkTemplate_Filtered2]) = 1, "Filter2 Only",
            "No Filters"
        )
)
ORDER BY Dim_WorkTemplates[Name]
```

### Example 5: Template Creation Timeline
```dax
EVALUATE
SELECTCOLUMNS(
    TOPN(
        50,
        Dim_WorkTemplates,
        Dim_WorkTemplates[Created_Date],
        DESC
    ),
    "Template_Name", Dim_WorkTemplates[Name],
    "Version", Dim_WorkTemplates[Version],
    "Identifier", Dim_WorkTemplates[Identifier],
    "Published", Dim_WorkTemplates[Is_Published],
    "Created_Date", FORMAT(Dim_WorkTemplates[Created_Date], "yyyy-MM-dd HH:mm"),
    "Days_Old", DATEDIFF(Dim_WorkTemplates[Created_Date], TODAY(), DAY)
)
```

### Example 6: Templates Without Template Links (Edge Case Check)
```dax
EVALUATE
// This query should return no results due to WHERE Template_Link IS NOT NULL filter
FILTER(
    SELECTCOLUMNS(
        Dim_WorkTemplates,
        "Template_Id", Dim_WorkTemplates[Id],
        "Template_Name", Dim_WorkTemplates[Name],
        "Template_Link", Dim_WorkTemplates[Template_Link]
    ),
    ISBLANK(Dim_WorkTemplates[Template_Link])
)
```

---

## Data Model Pattern

### Work Template Catalog Pattern
`Dim_WorkTemplates` serves as the master catalog of structured data collection templates used in the Obzervr application. Work templates define what information should be captured during work assignments and how it should be organized.

**Template Structure Hierarchy**:
Work templates are composed of a hierarchical structure of fragments:
1. **Work Template** (this table): Top-level container defining the entire form/workflow
2. **Group Fragments**: Major sections grouping related content (documented in `Dim_Fragments`)
3. **Section Fragments**: Subsections within groups (documented in `Dim_Fragments`)
4. **Field Fragments**: Individual data collection fields (documented in `Dim_Fragments`)

The detailed structure is stored in `Fact_Fragment_Details`, which links to this template catalog.

**Template Versioning**:
The `Version` column enables version control for templates:
- Multiple versions of the same template can coexist (same `Name`, different `Version`)
- `Identifier` provides a stable reference across versions
- `Is_Published` determines which versions are available for active use
- Historical assignments retain links to the template version used at assignment time

**Is_Published Status**:
- **TRUE**: Template is published and available for assignment creation
- **FALSE**: Template is in draft, archived, or deprecated state (not available for new assignments)

Published status enables:
- Draft template development without affecting active assignments
- Template retirement while maintaining historical assignment links
- A/B testing of template variants

**Template_Link Column**:
The `Template_Link` column contains a URL or identifier that links to the full template definition or editing interface. The WHERE clause filtering (`Template_Link IS NOT NULL`) ensures only templates with valid links are included, suggesting:
- Incomplete or orphaned template records are excluded
- Templates must have an accessible definition to be useful
- The link may point to the Obzervr template editor or external template storage

**JSON Structure**:
The `Json` column stores the template's complete structure in JSON format. This enables:
- Full template export/import
- Template structure analysis without parsing fragment tables
- Integration with external systems expecting JSON schemas
- Backup of template definitions

**Filtered Dimension Pattern**:
The two calculated columns (`Is_WorkTemplate_Filtered1`, `Is_WorkTemplate_Filtered2`) implement a filtered dimension pattern:
- **Purpose**: Indicate whether templates match specific business rules or subset criteria
- **Implementation**: LOOKUPVALUE against filtered versions of the same dimension
- **Use Cases**:
  - Template availability based on user permissions
  - Site-specific or department-specific template filtering
  - Template categorization for different work types
  - Conditional template visibility in reports

The binary flags (0/1 instead of TRUE/FALSE) suggest these are used in measures or visuals expecting numeric values.

**Example Scenario - Mining Pre-Start Inspection Template**:

**Template Record**:
- **Id**: 101
- **Name**: "Haul Truck Pre-Start Inspection"
- **Identifier**: "TMPL-HS-PS-001"
- **Version**: "2.1"
- **Is_Published**: TRUE
- **Template_Link**: "https://obzervr.app/templates/edit/101"
- **Fragment_Type**: "WorkTemplate"

**Template Structure** (in `Fact_Fragment_Details`):
```
Work Template: "Haul Truck Pre-Start Inspection"
├── Group: "Safety Checks"
│   ├── Section: "Personal Protective Equipment"
│   │   ├── Field: "Hard Hat Worn" (Boolean)
│   │   ├── Field: "Safety Boots Worn" (Boolean)
│   │   └── Field: "High-Vis Vest Worn" (Boolean)
│   └── Section: "Work Area Safety"
│       ├── Field: "Area Clear of Hazards" (Boolean)
│       └── Field: "Emergency Exits Clear" (Boolean)
├── Group: "Vehicle Inspection"
│   ├── Section: "Engine Compartment"
│   │   ├── Field: "Engine Oil Level" (Number, UOM: Liters, Range: 7-10)
│   │   ├── Field: "Coolant Level" (Number, UOM: %, Range: 80-100)
│   │   └── Field: "Hydraulic Fluid" (Boolean - OK/Not OK)
│   └── Section: "Tires"
│       ├── Field: "Front Left Tire Pressure" (Number, UOM: psi, Range: 90-100)
│       ├── Field: "Front Right Tire Pressure" (Number, UOM: psi, Range: 90-100)
│       └── Field: "Tire Damage Observed" (Boolean)
└── Group: "Sign-Off"
    └── Section: "Approval"
        ├── Field: "Inspector Signature" (Signature)
        ├── Field: "Inspection Time" (DateTime)
        └── Field: "Additional Comments" (Text)
```

**Version History**:
- Version 1.0: Original template (5 groups, 20 fields)
- Version 1.5: Added coolant level check (6 groups, 21 fields)
- Version 2.0: Restructured safety checks (6 groups, 23 fields)
- Version 2.1: Added tire pressure ranges (current, published)

Only version 2.1 has `Is_Published = TRUE`, but historical assignments may still reference versions 1.0-2.0.

**Filtered Dimension Usage**:
- `Is_WorkTemplate_Filtered1 = 1`: Template available to "Field Operators" role
- `Is_WorkTemplate_Filtered2 = 1`: Template available for "Mobile Equipment" asset category

---

## Related Documentation
- **ERD_06_Templates_Fragments.md** - ERD diagram showing template and fragment relationship context
- **Dim_Fragments.md** - Fragment catalog (groups, sections, fields)
- **Fact_Fragment_Details.md** - Detailed template structure linking fragments to templates
- **Assignments.md** - Assignments using work templates

---

## Change History
| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-15 | Auto-generated | Initial documentation from TMDL metadata |

---

## Notes
- **Fragment Type Filtering**: The WHERE clause filters to `Fragment_Type = 'WorkTemplate'`, suggesting the source table contains multiple fragment types but only work templates (top-level) are loaded into this dimension.
- **Template_Link Required**: The WHERE clause filters to `Template_Link IS NOT NULL`, excluding templates without valid links, likely orphaned or incomplete records.
- **Calculated Filter Flags**: The two `Is_WorkTemplate_Filtered` columns use LOOKUPVALUE against filtered dimension tables not documented in provided TMDL files, suggesting these are calculated tables or parameter-driven subsets.
- **Version Control**: Multiple rows with the same `Name` but different `Version` values enable template version history and evolution tracking.
- **Published Status**: Only templates with `Is_Published = TRUE` should be available for new assignment creation, though historical assignments retain links to their original template versions.
- **JSON Storage**: The `Json` column provides a complete self-contained template definition, enabling export, backup, and integration scenarios.
- **Identifier Stability**: The `Identifier` column provides a stable reference across template versions, while `Id` (primary key) is unique per version.
- **Template_Link Format**: The format of Template_Link is not specified but likely contains URLs to the Obzervr template editor or external template management systems.
- **Tenant Isolation**: Templates are filtered by Tenant_Id in Power Query, maintaining multi-tenant isolation.
- **LastLoaded Timestamp**: Enables ETL monitoring and refresh validation.
- **Fragment Hierarchy**: This table represents the top level of a 3-level hierarchy: WorkTemplate → Fragments (Groups/Sections/Fields) → Fragment Details.
- **Template Retirement**: Rather than deleting templates, setting `Is_Published = FALSE` retires them while maintaining referential integrity for historical assignments.
- **Numeric Filter Flags**: The calculated columns return 0/1 instead of TRUE/FALSE booleans, likely for easier use in measures (SUM) or visuals expecting numeric values.
