# AssignmentPoint_Attributes

## Table Overview
`AssignmentPoint_Attributes` is an incremental refresh table implementing an Entity-Attribute-Value (EAV) pattern to store flexible, dynamic metadata about assignment points. Each row represents a single attribute name-value pair associated with an assignment point, allowing custom properties to be defined without schema changes.

This table enables tenant-specific or custom attributes to be attached to assignment points (equipment, locations, assets) for enhanced filtering, reporting, and contextual analysis.

**Current Status**: Incremental Refresh enabled with 20-year rolling window and 1-year increments, using polling refresh expression based on `Last_Updated`.

---

## Specifications
- **Source**: Data warehouse table (EAV pattern)
- **Row Count**: Variable (depends on custom attribute usage per tenant)
- **Grain**: One row per attribute-value pair per assignment point
- **Primary Key**: `Id`
- **Incremental Refresh**: 20 years rolling, 1-year increments, polling on `Last_Updated`
- **Partitioning Strategy**: Incremental refresh with polling expression
- **Source Columns**: 8
- **Calculated Columns**: 0
- **Parameter Dependency**: `EnableAssignmentPointAttributes` (table load conditional)

---

## Column Specifications

| Column Name | Data Type | Format | Nullable | Hidden | Description |
|------------|-----------|--------|----------|--------|-------------|
| Attribute_Name | String | | No | No | Name of the custom attribute (e.g., "Equipment Type", "Cost Center", "Building") |
| Attribute_Value | String | | No | No | Value assigned to this attribute for the assignment point |
| Attribute_GroupName | String | | Yes | No | Logical grouping for related attributes (e.g., "Equipment Details", "Location Info") |
| AssignmentPoint_Id | Int64 | | No | No | Foreign key to the assignment point this attribute belongs to |
| Created_Date | DateTime | yyyy-MM-dd HH:mm:ss | No | No | Timestamp when this attribute was first created |
| Last_Updated | DateTime | yyyy-MM-dd HH:mm:ss | No | No | Timestamp of last modification (used for polling refresh) |
| Tenant_Id | Int64 | | No | No | Foreign key to tenant dimension |
| Id | Int64 | | No | No | Primary key for the attribute record |

---

## Calculated Columns
None. This table uses only source columns from the data warehouse.

---

## Relationships

### Outbound Relationships
| To Table | From Column(s) | To Column(s) | Cardinality | Cross Filter | Relationship ID |
|----------|---------------|--------------|-------------|--------------|-----------------|
| `Dim_AssignmentPoints` | AssignmentPoint_Id | AssignmentPoint_Id | Many-to-One | Single | (Assignment point context) |
| `Dim_Tenant` | Tenant_Id | Tenant_Id | Many-to-One | Single | (Tenant context) |

### Inbound Relationships
None. This is a leaf table in the relationship hierarchy.

---

## Power Query M Source

```m
let
    Source = if EnableAssignmentPointAttributes = true then
        Value.NativeQuery(
            Obzervr_DataWarehouse,
            "SELECT 
                Attribute_Name,
                Attribute_Value,
                Attribute_GroupName,
                AssignmentPoint_Id,
                Created_Date,
                Last_Updated,
                TenantId,
                Id
            FROM [dbo].[AssignmentPoint_Attributes]"
        )
    else
        #table(
            {"Attribute_Name", "Attribute_Value", "Attribute_GroupName", "AssignmentPoint_Id", "Created_Date", "Last_Updated", "Tenant_Id", "Id"},
            {}
        ),
    
    #"Filtered Tenant" = Table.SelectRows(
        Source,
        each [TenantId] = TenantId
    ),
    #"Renamed TenantId" = Table.RenameColumns(
        #"Filtered Tenant",
        {{"TenantId", "Tenant_Id"}}
    ),
    #"Changed Types" = Table.TransformColumnTypes(
        #"Renamed TenantId",
        {
            {"Attribute_Name", type text},
            {"Attribute_Value", type text},
            {"Attribute_GroupName", type text},
            {"AssignmentPoint_Id", Int64.Type},
            {"Created_Date", type datetime},
            {"Last_Updated", type datetime},
            {"Tenant_Id", Int64.Type},
            {"Id", Int64.Type}
        }
    )
in
    #"Changed Types"
```

### Incremental Refresh Configuration
```m
// Polling expression to detect new/updated attributes
Source = if EnableAssignmentPointAttributes = true then
    Value.NativeQuery(
        Obzervr_DataWarehouse,
        "SELECT MAX(Last_Updated) as MaxLastUpdated 
         FROM [dbo].[AssignmentPoint_Attributes]"
    )[MaxLastUpdated]?
    ?? #datetime(1901, 1, 1, 0, 0, 0)
else
    #datetime(1901, 1, 1, 0, 0, 0)
```

**Incremental Refresh Policy**:
- **Rolling Window**: 20 years
- **Incremental Period**: 1 year
- **Polling**: Enabled - checks MAX(Last_Updated) to detect changes
- **Archive**: Older than 20 years automatically removed

---

## DAX Query Patterns

### Example 1: Assignment Points by Attribute Type
```dax
EVALUATE
SUMMARIZECOLUMNS(
    AssignmentPoint_Attributes[Attribute_Name],
    "Unique_Values", DISTINCTCOUNT(AssignmentPoint_Attributes[Attribute_Value]),
    "Assignment_Point_Count", DISTINCTCOUNT(AssignmentPoint_Attributes[AssignmentPoint_Id]),
    "Total_Attribute_Records", COUNTROWS(AssignmentPoint_Attributes)
)
ORDER BY [Assignment_Point_Count] DESC
```

### Example 2: Attribute Values for Specific Assignment Point
```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        AssignmentPoint_Attributes,
        "Assignment_Point", RELATED(Dim_AssignmentPoints[AssignmentPoint_Name]),
        "Group", AssignmentPoint_Attributes[Attribute_GroupName],
        "Attribute", AssignmentPoint_Attributes[Attribute_Name],
        "Value", AssignmentPoint_Attributes[Attribute_Value],
        "Last_Updated", AssignmentPoint_Attributes[Last_Updated]
    ),
    AssignmentPoint_Attributes[AssignmentPoint_Id] = 12345
)
ORDER BY AssignmentPoint_Attributes[Attribute_GroupName], AssignmentPoint_Attributes[Attribute_Name]
```

### Example 3: Attribute Group Distribution
```dax
EVALUATE
SUMMARIZECOLUMNS(
    AssignmentPoint_Attributes[Attribute_GroupName],
    "Attribute_Types", DISTINCTCOUNT(AssignmentPoint_Attributes[Attribute_Name]),
    "Assignment_Points_Tagged", DISTINCTCOUNT(AssignmentPoint_Attributes[AssignmentPoint_Id]),
    "Total_Attributes", COUNTROWS(AssignmentPoint_Attributes)
)
ORDER BY [Total_Attributes] DESC
```

### Example 4: Recently Modified Attributes
```dax
EVALUATE
TOPN(
    50,
    SELECTCOLUMNS(
        AssignmentPoint_Attributes,
        "Assignment_Point", RELATED(Dim_AssignmentPoints[AssignmentPoint_Name]),
        "Attribute", AssignmentPoint_Attributes[Attribute_Name],
        "Value", AssignmentPoint_Attributes[Attribute_Value],
        "Group", AssignmentPoint_Attributes[Attribute_GroupName],
        "Last_Updated", AssignmentPoint_Attributes[Last_Updated]
    ),
    AssignmentPoint_Attributes[Last_Updated],
    DESC
)
```

### Example 5: Assignment Points with Multiple Attributes
```dax
EVALUATE
FILTER(
    SUMMARIZECOLUMNS(
        Dim_AssignmentPoints[AssignmentPoint_Name],
        "Attribute_Count", COUNTROWS(AssignmentPoint_Attributes),
        "Attribute_Groups", DISTINCTCOUNT(AssignmentPoint_Attributes[Attribute_GroupName]),
        "Last_Modified", MAX(AssignmentPoint_Attributes[Last_Updated])
    ),
    [Attribute_Count] > 1
)
ORDER BY [Attribute_Count] DESC
```

### Example 6: Specific Attribute Value Search
```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        AssignmentPoint_Attributes,
        "Assignment_Point_Id", AssignmentPoint_Attributes[AssignmentPoint_Id],
        "Assignment_Point_Name", RELATED(Dim_AssignmentPoints[AssignmentPoint_Name]),
        "Attribute_Name", AssignmentPoint_Attributes[Attribute_Name],
        "Attribute_Value", AssignmentPoint_Attributes[Attribute_Value],
        "Group", AssignmentPoint_Attributes[Attribute_GroupName]
    ),
    AssignmentPoint_Attributes[Attribute_Name] = "Equipment Type"
    && AssignmentPoint_Attributes[Attribute_Value] = "Haul Truck"
)
```

---

## Data Model Pattern

### Entity-Attribute-Value (EAV) Pattern
`AssignmentPoint_Attributes` implements the EAV pattern, which stores three key pieces of information per row:
- **Entity**: The `AssignmentPoint_Id` identifies which assignment point owns this attribute
- **Attribute**: The `Attribute_Name` defines what property is being stored (e.g., "Cost Center", "Building", "Equipment Class")
- **Value**: The `Attribute_Value` contains the actual value for that attribute

**Advantages**:
- **Schema Flexibility**: New attributes can be added without modifying table structure
- **Sparse Data Handling**: Assignment points only store attributes that apply to them (no NULL columns for unused attributes)
- **Tenant Customization**: Different tenants can define custom attributes specific to their business needs

**Trade-offs**:
- **Querying Complexity**: Attribute-specific filters require more complex DAX compared to fixed columns
- **Type Safety**: All values stored as strings; numeric/date conversion required in measures
- **Performance Considerations**: Pivoting EAV data into columnar format for reports requires aggregation

**Attribute_GroupName Usage**: Groups related attributes together for organizational purposes. Example groupings:
- **"Equipment Details"**: Equipment Type, Model, Serial Number, Manufacturer
- **"Location Info"**: Building, Floor, Zone, GPS Coordinates
- **"Cost Tracking"**: Cost Center, Department, Budget Code
- **"Maintenance"**: Last Service Date, Next Service Due, Warranty Expiry

**Example Scenario - Mining Equipment**:

Assignment Point: "Haul Truck HT-001" (AssignmentPoint_Id = 12345)

Attributes stored in this table:
| Attribute_Name | Attribute_Value | Attribute_GroupName |
|---------------|----------------|---------------------|
| Equipment Type | Haul Truck | Equipment Details |
| Model | CAT 789D | Equipment Details |
| Capacity (Tonnes) | 181 | Equipment Details |
| Cost Center | CC-MINING-001 | Cost Tracking |
| Operating Area | Pit 3 | Location Info |
| Tire Type | 40.00R57 | Maintenance |
| Last Major Service | 2024-12-15 | Maintenance |

This EAV structure allows flexible attribute definition without requiring schema changes when new attributes are needed (e.g., adding "Fuel Type" or "GPS Unit ID").

---

## Related Documentation
- **ERD_03_Field_Measurements.md** - ERD diagram showing relationship context
- **Dim_AssignmentPoints.md** - Parent dimension table for assignment points
- **Assignments.md** - Assignment point usage in work assignments
- **TimeSeries.md** - Assignment point usage in timeseries tracking

---

## Change History
| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-15 | Auto-generated | Initial documentation from TMDL metadata |

---

## Notes
- **EAV Pattern**: This table uses the Entity-Attribute-Value pattern to store flexible metadata about assignment points. Each row represents one attribute-value pair rather than having separate columns for each possible attribute.
- **Polling Refresh**: The incremental refresh uses a polling expression that checks MAX(Last_Updated) to detect when new or modified attributes exist. If no attributes exist, defaults to 1901-01-01.
- **Parameter Dependency**: The table only loads data when `EnableAssignmentPointAttributes` parameter is true. When false, an empty table with correct schema is created.
- **20-Year Window**: The 20-year rolling window is significantly longer than most fact tables, suggesting attributes are relatively static and historical attribute values may be needed for compliance or audit purposes.
- **String Values**: All attribute values are stored as strings. DAX measures must convert to numeric/date types when needed using VALUE(), DATEVALUE(), or similar functions.
- **Attribute Pivoting**: To create fixed columns from EAV attributes for reporting, use calculated columns or measures with LOOKUPVALUE() or CALCULATE(MAX(...), FILTER(...)) patterns.
- **Group-Based Organization**: Use `Attribute_GroupName` to organize attributes in reports (e.g., create separate sections for Equipment Details, Location Info, Cost Tracking).
- **No Soft Deletes**: This table does not have an IsDeleted flag. Attributes are either present or not present.
- **Tenant Filtering**: Like other tables, this is filtered to the current tenant in Power Query, preventing cross-tenant attribute visibility.
- **Incremental Refresh Performance**: The polling refresh approach means the model checks for attribute changes even if the MAX(Last_Updated) value hasn't changed. This ensures attribute deletions (which wouldn't change MAX) trigger refresh.
