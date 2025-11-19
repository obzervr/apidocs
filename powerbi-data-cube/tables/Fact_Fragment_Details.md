# Fact_Fragment_Details

## Table Overview
`Fact_Fragment_Details` is an incremental refresh fact table that stores the detailed hierarchical structure of work templates, breaking down templates into their constituent groups, sections, and fields. Each row represents a single field within the template's 3-level hierarchy (Group → Section → Field) with complete metadata including sequence numbers, field properties, and boundaries.

This table materializes the composition of work templates from fragment building blocks, enabling template structure analysis and field-level reporting.

**Current Status**: Incremental Refresh enabled with 5-year rolling window and 1-month increments based on `LoadDate`. Table load conditional on `EnableTemplateFragmentData` parameter.

---

## Specifications
- **Source**: `VW_FragmentDetails` view
- **Row Count**: High volume (every field in every template × versions)
- **Grain**: One row per field within a template's group-section-field hierarchy
- **Primary Key**: Composite (WorkTemplate_Id + Group_Id + Section_Id + Field_Id)
- **Incremental Refresh**: 5 years rolling, 1-month increments, filtered on `LoadDate`
- **Partitioning Strategy**: Incremental refresh by load date
- **Source Columns**: 32
- **Calculated Columns**: 0
- **Parameter Dependency**: `EnableTemplateFragmentData` (table load conditional)

---

## Column Specifications

| Column Name | Data Type | Format | Nullable | Hidden | Description |
|------------|-----------|--------|----------|--------|-------------|
| Tenant_Id | Int64 | | No | No | Foreign key to tenant dimension |
| WorkTemplate_Name | String | | No | No | Display name of the work template |
| WorkTemplate_Id | Int64 | | No | No | Foreign key to work template dimension |
| Phase | String | | Yes | No | Template phase or stage classification |
| Group_Name | String | | Yes | No | Name of the group fragment (top-level hierarchy) |
| Group_Identifier | String | | Yes | No | Unique identifier for the group fragment |
| Group_Id | Int64 | | Yes | No | Foreign key to group fragment in Dim_Fragments |
| Group_TemplateLink | String | | Yes | No | URL or identifier for group fragment definition |
| Group_SequenceNo | Int64 | | Yes | No | Sequence number defining group order within template |
| Section_Id | Int64 | | Yes | No | Foreign key to section fragment in Dim_Fragments |
| Section_TemplateLink | String | | Yes | No | URL or identifier for section fragment definition |
| Section_Name | String | | Yes | No | Name of the section fragment (mid-level hierarchy) |
| Section_Identifier | String | | Yes | No | Unique identifier for the section fragment |
| Section_SequenceNo | Int64 | | Yes | No | Sequence number defining section order within group |
| Field_Id | Int64 | | No | No | Foreign key to field fragment in Dim_Fragments |
| Field_Name | String | | No | No | Name of the field fragment (leaf-level) |
| Field_Identifier | String | | No | No | Unique identifier for the field fragment |
| Field_SequenceNo | Int64 | | Yes | No | Sequence number defining field order within section |
| Field_Type | String | | Yes | No | Data type classification (Bool, Date, Time, Text, Number, Location, Signature, Attachment, MultiSelect) |
| Preface | String | | Yes | No | Text displayed before the field |
| Postface | String | | Yes | No | Text displayed after the field |
| Help_Text | String | | Yes | No | Help or instruction text for the field |
| UOM | String | | Yes | No | Unit of measurement for numeric fields (e.g., "kg", "°C", "psi") |
| Is_ReadOnly | String | | Yes | No | "true"/"false" string indicating if field is read-only |
| Is_Required | String | | Yes | No | "true"/"false" string indicating if field is mandatory |
| Number_Of_Attachments | Int64 | | Yes | No | Count of attachments allowed/associated with field |
| Number_Of_InlinePhotos | Int64 | | Yes | No | Count of inline photos allowed/associated with field |
| Lower_Boundary | Double | | Yes | No | Minimum acceptable value for numeric fields |
| Upper_Boundary | Double | | Yes | No | Maximum acceptable value for numeric fields |
| LoadDate | DateTime | yyyy-MM-dd HH:mm:ss | No | No | ETL load timestamp (incremental refresh key) |
| Version_Id | Int64 | | Yes | No | Version identifier for template versioning |
| Template_Link | String | | Yes | No | URL or identifier for field template definition |

---

## Calculated Columns
None. This table uses only source columns from the view.

---

## Relationships

### Outbound Relationships
| To Table | From Column(s) | To Column(s) | Cardinality | Cross Filter | Relationship ID |
|----------|---------------|--------------|-------------|--------------|-----------------|
| `Dim_Tenant` | Tenant_Id | Tenant_Id | Many-to-One | Single | (Tenant context) |
| `Dim_WorkTemplates` | WorkTemplate_Id | Id | Many-to-One | Single | (Work template context) |
| `Dim_Fragments` | Group_Id | Id | Many-to-One | Single | (Group fragment context) |
| `Dim_Fragments` | Section_Id | Id | Many-to-One | Single | (Section fragment context - inactive) |
| `Dim_Fragments` | Field_Id | Id | Many-to-One | Single | (Field fragment context - inactive) |

### Inbound Relationships
None. This is a leaf table in the relationship hierarchy.

---

## Power Query M Source

```m
let
    Source = if EnableTemplateFragmentData = true then
        Value.NativeQuery(
            Obzervr_DataWarehouse,
            "SELECT 
                Tenant_Id,
                WorkTemplate_Name,
                WorkTemplate_Id,
                Phase,
                Group_Name,
                Group_Identifier,
                Group_Id,
                Group_TemplateLink,
                Group_SequenceNo,
                Section_Id,
                Section_TemplateLink,
                Section_Name,
                Section_Identifier,
                Section_SequenceNo,
                Field_Id,
                Field_Name,
                Field_Identifier,
                Field_SequenceNo,
                Field_Type,
                Preface,
                Postface,
                Help_Text,
                UOM,
                Is_ReadOnly,
                Is_Required,
                Number_Of_Attachments,
                Number_Of_InlinePhotos,
                Lower_Boundary,
                Upper_Boundary,
                LoadDate,
                Version_Id,
                Template_Link
            FROM [dbo].[VW_FragmentDetails]"
        )
    else
        #table(
            {
                "Tenant_Id", "WorkTemplate_Name", "WorkTemplate_Id", "Phase",
                "Group_Name", "Group_Identifier", "Group_Id", "Group_TemplateLink", "Group_SequenceNo",
                "Section_Id", "Section_TemplateLink", "Section_Name", "Section_Identifier", "Section_SequenceNo",
                "Field_Id", "Field_Name", "Field_Identifier", "Field_SequenceNo", "Field_Type",
                "Preface", "Postface", "Help_Text", "UOM", "Is_ReadOnly", "Is_Required",
                "Number_Of_Attachments", "Number_Of_InlinePhotos", "Lower_Boundary", "Upper_Boundary",
                "LoadDate", "Version_Id", "Template_Link"
            },
            {}
        ),
    
    #"Filtered Tenant" = Table.SelectRows(
        Source,
        each [Tenant_Id] = TenantId
    ),
    #"Changed Types" = Table.TransformColumnTypes(
        #"Filtered Tenant",
        {
            {"Tenant_Id", Int64.Type},
            {"WorkTemplate_Name", type text},
            {"WorkTemplate_Id", Int64.Type},
            {"Phase", type text},
            {"Group_Name", type text},
            {"Group_Identifier", type text},
            {"Group_Id", Int64.Type},
            {"Group_TemplateLink", type text},
            {"Group_SequenceNo", Int64.Type},
            {"Section_Id", Int64.Type},
            {"Section_TemplateLink", type text},
            {"Section_Name", type text},
            {"Section_Identifier", type text},
            {"Section_SequenceNo", Int64.Type},
            {"Field_Id", Int64.Type},
            {"Field_Name", type text},
            {"Field_Identifier", type text},
            {"Field_SequenceNo", Int64.Type},
            {"Field_Type", type text},
            {"Preface", type text},
            {"Postface", type text},
            {"Help_Text", type text},
            {"UOM", type text},
            {"Is_ReadOnly", type text},
            {"Is_Required", type text},
            {"Number_Of_Attachments", Int64.Type},
            {"Number_Of_InlinePhotos", Int64.Type},
            {"Lower_Boundary", type number},
            {"Upper_Boundary", type number},
            {"LoadDate", type datetime},
            {"Version_Id", Int64.Type},
            {"Template_Link", type text}
        }
    ),
    #"Filtered by Incremental Refresh" = Table.SelectRows(
        #"Changed Types",
        each [LoadDate] >= RangeStart and [LoadDate] < RangeEnd
    )
in
    #"Filtered by Incremental Refresh"
```

---

## DAX Query Patterns

### Example 1: Template Structure Overview
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Fact_Fragment_Details[WorkTemplate_Name],
    "Group_Count", DISTINCTCOUNT(Fact_Fragment_Details[Group_Id]),
    "Section_Count", DISTINCTCOUNT(Fact_Fragment_Details[Section_Id]),
    "Field_Count", DISTINCTCOUNT(Fact_Fragment_Details[Field_Id]),
    "Required_Field_Count", COUNTROWS(
        FILTER(Fact_Fragment_Details, Fact_Fragment_Details[Is_Required] = "true")
    )
)
ORDER BY [Field_Count] DESC
```

### Example 2: Field Type Distribution by Template
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Fact_Fragment_Details[WorkTemplate_Name],
    Fact_Fragment_Details[Field_Type],
    "Field_Count", COUNTROWS(Fact_Fragment_Details),
    "Required_Count", COUNTROWS(
        FILTER(Fact_Fragment_Details, Fact_Fragment_Details[Is_Required] = "true")
    ),
    "With_Boundaries", COUNTROWS(
        FILTER(
            Fact_Fragment_Details,
            NOT ISBLANK(Fact_Fragment_Details[Lower_Boundary])
            || NOT ISBLANK(Fact_Fragment_Details[Upper_Boundary])
        )
    )
)
ORDER BY Fact_Fragment_Details[WorkTemplate_Name], [Field_Count] DESC
```

### Example 3: Template Hierarchy Browser
```dax
EVALUATE
SELECTCOLUMNS(
    FILTER(
        Fact_Fragment_Details,
        Fact_Fragment_Details[WorkTemplate_Name] = "Haul Truck Pre-Start Inspection"
    ),
    "Hierarchy", 
        Fact_Fragment_Details[Group_SequenceNo] & " - " & Fact_Fragment_Details[Group_Name]
        & " > " & Fact_Fragment_Details[Section_SequenceNo] & " - " & Fact_Fragment_Details[Section_Name]
        & " > " & Fact_Fragment_Details[Field_SequenceNo] & " - " & Fact_Fragment_Details[Field_Name],
    "Field_Type", Fact_Fragment_Details[Field_Type],
    "Required", Fact_Fragment_Details[Is_Required],
    "UOM", Fact_Fragment_Details[UOM],
    "Lower_Boundary", Fact_Fragment_Details[Lower_Boundary],
    "Upper_Boundary", Fact_Fragment_Details[Upper_Boundary]
)
ORDER BY 
    Fact_Fragment_Details[Group_SequenceNo],
    Fact_Fragment_Details[Section_SequenceNo],
    Fact_Fragment_Details[Field_SequenceNo]
```

### Example 4: Required Fields Report
```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Fact_Fragment_Details,
        "Template", Fact_Fragment_Details[WorkTemplate_Name],
        "Group", Fact_Fragment_Details[Group_Name],
        "Section", Fact_Fragment_Details[Section_Name],
        "Field", Fact_Fragment_Details[Field_Name],
        "Field_Type", Fact_Fragment_Details[Field_Type],
        "Help_Text", Fact_Fragment_Details[Help_Text]
    ),
    Fact_Fragment_Details[Is_Required] = "true"
)
ORDER BY 
    Fact_Fragment_Details[WorkTemplate_Name],
    Fact_Fragment_Details[Group_SequenceNo],
    Fact_Fragment_Details[Section_SequenceNo],
    Fact_Fragment_Details[Field_SequenceNo]
```

### Example 5: Numeric Fields with Boundaries
```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Fact_Fragment_Details,
        "Template", Fact_Fragment_Details[WorkTemplate_Name],
        "Field_Name", Fact_Fragment_Details[Field_Name],
        "UOM", Fact_Fragment_Details[UOM],
        "Lower_Boundary", Fact_Fragment_Details[Lower_Boundary],
        "Upper_Boundary", Fact_Fragment_Details[Upper_Boundary],
        "Range_Width", Fact_Fragment_Details[Upper_Boundary] - Fact_Fragment_Details[Lower_Boundary],
        "Help_Text", Fact_Fragment_Details[Help_Text]
    ),
    Fact_Fragment_Details[Field_Type] = "Number"
    && NOT ISBLANK(Fact_Fragment_Details[Lower_Boundary])
    && NOT ISBLANK(Fact_Fragment_Details[Upper_Boundary])
)
ORDER BY Fact_Fragment_Details[WorkTemplate_Name], [Range_Width] DESC
```

### Example 6: Fragment Reuse Analysis
```dax
EVALUATE
ADDCOLUMNS(
    SUMMARIZE(
        Fact_Fragment_Details,
        Fact_Fragment_Details[Field_Name],
        Fact_Fragment_Details[Field_Identifier],
        Fact_Fragment_Details[Field_Type]
    ),
    "Used_In_Templates", CALCULATE(
        DISTINCTCOUNT(Fact_Fragment_Details[WorkTemplate_Id])
    ),
    "Used_In_Sections", CALCULATE(
        DISTINCTCOUNT(Fact_Fragment_Details[Section_Id])
    ),
    "Total_Occurrences", CALCULATE(
        COUNTROWS(Fact_Fragment_Details)
    )
)
ORDER BY [Used_In_Templates] DESC
```

---

## Data Model Pattern

### Template Composition Denormalized Hierarchy Pattern
`Fact_Fragment_Details` implements a denormalized hierarchy pattern that stores the complete 3-level template structure (Group → Section → Field) in a single flat table. This denormalization trades storage efficiency for query performance and simplicity.

**3-Level Hierarchy Structure**:
Each row represents the lowest level (field) but includes complete context from all parent levels:

```
Level 1: GROUP
├── Group_Id, Group_Name, Group_Identifier, Group_TemplateLink, Group_SequenceNo
│
└── Level 2: SECTION
    ├── Section_Id, Section_Name, Section_Identifier, Section_TemplateLink, Section_SequenceNo
    │
    └── Level 3: FIELD (this row's grain)
        └── Field_Id, Field_Name, Field_Identifier, Field_SequenceNo, Field_Type, properties
```

**Denormalization Benefits**:
- **Single Query**: Complete template structure available without joins
- **Performance**: Sequence-based ordering without recursive queries
- **Simplicity**: Hierarchy navigation using simple filters on sequence numbers
- **Reporting**: Easy aggregation at any hierarchy level (group, section, or field)

**Sequence Number Ordering**:
Three sequence number columns define display order:
- **Group_SequenceNo**: Order of groups within template (1, 2, 3, ...)
- **Section_SequenceNo**: Order of sections within each group (1, 2, 3, ...)
- **Field_SequenceNo**: Order of fields within each section (1, 2, 3, ...)

Combined ordering: `GROUP_SEQ.SECTION_SEQ.FIELD_SEQ` (e.g., 1.2.3 = Group 1, Section 2, Field 3)

**Field Properties**:
Each field row includes detailed metadata:
- **Field_Type**: Data type classification ("Bool", "Date", "Time", "Text", "Number", "Location", "Signature", "Attachment", "MultiSelect")
- **Validation**: Lower_Boundary, Upper_Boundary for numeric types
- **UI Elements**: Preface, Postface, Help_Text for display
- **Measurements**: UOM for numeric fields
- **Configuration**: Is_ReadOnly, Is_Required as string "true"/"false"
- **Attachments**: Number_Of_Attachments, Number_Of_InlinePhotos

**String Booleans** (Is_ReadOnly, Is_Required):
These fields use "true"/"false" strings instead of boolean types, likely for:
- Source system compatibility (JSON serialization)
- Easier filtering in slicers (text vs boolean display)
- NULL handling (empty string vs NULL vs FALSE)

**Phase Column**:
The `Phase` column provides optional template phase classification (e.g., "Pre-Start", "Operational", "Post-Operation"), enabling:
- Filtering templates by operational phase
- Phase-based reporting
- Multi-phase workflow templates

**Incremental Refresh Strategy**:
The 5-year rolling window with 1-month increments on `LoadDate` suggests:
- Template structures are versioned and reloaded monthly
- Historical template structures are maintained for 5 years
- Template evolution can be tracked over time
- LoadDate represents when the structure was captured, not when template was created

**Parameter-Based Loading**:
The `EnableTemplateFragmentData` parameter controls whether this table loads:
- **TRUE**: Full template structure data loaded
- **FALSE**: Empty table with schema only (no data)

This pattern enables:
- Deployment-specific configuration (disable for non-template-focused reports)
- Performance optimization (skip large table if not needed)
- Licensing or feature gating

**Multiple Relationships to Dim_Fragments**:
The table has three relationships to `Dim_Fragments`:
1. **Group_Id** → Dim_Fragments[Id] (active)
2. **Section_Id** → Dim_Fragments[Id] (inactive)
3. **Field_Id** → Dim_Fragments[Id] (inactive)

DAX measures must use USERELATIONSHIP to activate the correct relationship context.

**Example Scenario - Haul Truck Pre-Start Inspection Template Structure**:

| Group_Seq | Group_Name | Section_Seq | Section_Name | Field_Seq | Field_Name | Field_Type | Is_Required | Lower | Upper | UOM |
|-----------|------------|-------------|--------------|-----------|------------|------------|-------------|-------|-------|-----|
| 1 | Safety Checks | 1 | Personal Protective Equipment | 1 | Hard Hat Worn | Bool | true | | | |
| 1 | Safety Checks | 1 | Personal Protective Equipment | 2 | Safety Boots Worn | Bool | true | | | |
| 1 | Safety Checks | 1 | Personal Protective Equipment | 3 | High-Vis Vest Worn | Bool | true | | | |
| 1 | Safety Checks | 2 | Work Area Safety | 1 | Area Clear of Hazards | Bool | true | | | |
| 2 | Vehicle Inspection | 1 | Engine Compartment | 1 | Engine Oil Level | Number | true | 7 | 10 | L |
| 2 | Vehicle Inspection | 1 | Engine Compartment | 2 | Coolant Level | Number | true | 80 | 100 | % |
| 2 | Vehicle Inspection | 2 | Tires | 1 | Front Left Tire Pressure | Number | true | 90 | 100 | psi |
| 2 | Vehicle Inspection | 2 | Tires | 2 | Front Right Tire Pressure | Number | true | 90 | 100 | psi |
| 3 | Sign-Off | 1 | Approval | 1 | Inspector Signature | Signature | true | | | |
| 3 | Sign-Off | 1 | Approval | 2 | Inspection Time | Date | true | | | |
| 3 | Sign-Off | 1 | Approval | 3 | Additional Comments | Text | false | | | |

This denormalized structure enables queries like:
- "List all required fields" (filter Is_Required = "true")
- "Count sections per group" (DISTINCTCOUNT(Section_Id) by Group_Id)
- "Show fields in sequence" (ORDER BY Group_Seq, Section_Seq, Field_Seq)
- "Find numeric fields with boundaries" (filter Field_Type = "Number" AND boundaries NOT NULL)

---

## Related Documentation
- **ERD_06_Templates_Fragments.md** - ERD diagram showing template and fragment relationship context
- **Dim_WorkTemplates.md** - Work template catalog
- **Dim_Fragments.md** - Fragment catalog (groups, sections, fields)
- **TimeSeries_FieldMeasurements.md** - Captured field measurement data using these template structures

---

## Change History
| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-15 | Auto-generated | Initial documentation from TMDL metadata |

---

## Notes
- **Denormalized Hierarchy**: This table stores a 3-level hierarchy (Group → Section → Field) in denormalized format, with all parent context repeated for each field row.
- **Grain**: The table's grain is at the field level - one row per field within a template's structure.
- **Sequence-Based Ordering**: Three sequence number columns (Group_SequenceNo, Section_SequenceNo, Field_SequenceNo) define display order without requiring recursive queries.
- **String Booleans**: Is_ReadOnly and Is_Required use "true"/"false" strings instead of boolean types, likely for source system compatibility.
- **Parameter-Based Loading**: The table only loads when `EnableTemplateFragmentData = true`, creating empty schema when false.
- **Incremental Refresh**: 5-year rolling window with 1-month increments on LoadDate suggests monthly captures of template structures for historical tracking.
- **Multiple Relationships**: Three relationships to Dim_Fragments (Group_Id, Section_Id, Field_Id) with only one active; use USERELATIONSHIP in measures for section/field context.
- **Field Type Classification**: Field_Type uses the same classification as TimeSeries_FieldMeasurements (Bool, Date, Time, Text, Number, Location, Signature, Attachment, MultiSelect).
- **Boundary Validation**: Lower_Boundary and Upper_Boundary define acceptable ranges for numeric fields, matching boundary enforcement in TimeSeries_FieldMeasurements and Assignment_FieldMeasurement_Exceptions.
- **UOM Storage**: Unit of measurement stored at field level rather than measurement level, defining expected UOM for all readings of this field.
- **Preface/Postface**: UI text elements that appear before/after the field in the Obzervr application interface.
- **Help_Text**: Instruction text displayed to guide field technicians when capturing measurements.
- **Attachment Configuration**: Number_Of_Attachments and Number_Of_InlinePhotos define attachment capabilities at template design time.
- **Template_Link**: Field-level template link may point to field fragment editor or detailed field configuration interface.
- **Version_Id**: Enables template version tracking at the structure level (matching Version column in Dim_WorkTemplates/Dim_Fragments).
- **Phase Classification**: Optional phase grouping enables multi-phase workflows (e.g., Pre-Start → Operational → Post-Operation).
- **LoadDate vs Created_Date**: LoadDate represents when structure was captured/loaded, while Created_Date (from source) would represent original creation.
- **Tenant Isolation**: Standard tenant filtering applied in Power Query.
- **NULL Handling**: Many columns are nullable, reflecting optional template features (e.g., not all fields have boundaries, UOMs, or help text).
- **Read-Only Fields**: Is_ReadOnly = "true" indicates display-only fields (e.g., calculated values, reference data) that cannot be edited during assignment execution.
- **Required Fields**: Is_Required = "true" enforces mandatory data collection, preventing assignment completion until all required fields are populated.
