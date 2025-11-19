# Assignment_Tags

## Table Overview
`Assignment_Tags` is a simple key-value pair table that provides flexible tagging capabilities for assignments. Each row represents a single tag applied to an assignment, enabling custom classification, categorization, and filtering beyond the fixed schema columns.

This table supports dynamic metadata attachment to assignments without requiring schema changes, similar to the EAV pattern used in `AssignmentPoint_Attributes`.

**Current Status**: Standard import table (no incremental refresh).

---

## Specifications
- **Source**: `FactAssignmentTags` table
- **Row Count**: Variable (depends on tagging usage per tenant)
- **Grain**: One row per key-value pair per assignment
- **Primary Key**: Composite (Assignment_Id + Key)
- **Incremental Refresh**: Not enabled
- **Partitioning Strategy**: Standard import
- **Source Columns**: 3
- **Calculated Columns**: 0
- **Filtering**: IsDeleted = false AND IsVisible = true

---

## Column Specifications

| Column Name | Data Type | Format | Nullable | Hidden | Description |
|------------|-----------|--------|----------|--------|-------------|
| Assignment_Id | Int64 | | No | No | Foreign key to the assignment being tagged |
| Key | String | | No | No | Tag name or category (e.g., "Priority", "Location", "Equipment Type") |
| Value | String | | No | No | Tag value assigned to this key for this assignment |

---

## Calculated Columns
None. This table uses only source columns from the fact table.

---

## Relationships

### Outbound Relationships
| To Table | From Column(s) | To Column(s) | Cardinality | Cross Filter | Relationship ID |
|----------|---------------|--------------|-------------|--------------|-----------------|
| `Assignments` | Assignment_Id | Assignment_Id | Many-to-One | Single | (Assignment context) |

### Inbound Relationships
None. This is a leaf table in the relationship hierarchy.

---

## Power Query M Source

```m
let
    Source = Value.NativeQuery(
        Obzervr_DataWarehouse,
        "SELECT 
            Assignment_Id,
            [Key],
            Value
        FROM [dbo].[FactAssignmentTags]
        WHERE IsDeleted = 0 
          AND IsVisible = 1"
    ),
    #"Filtered Tenant" = Table.SelectRows(
        Source,
        each [TenantId] = TenantId
    ),
    #"Removed TenantId Column" = Table.RemoveColumns(
        #"Filtered Tenant",
        {"TenantId"}
    ),
    #"Changed Types" = Table.TransformColumnTypes(
        #"Removed TenantId Column",
        {
            {"Assignment_Id", Int64.Type},
            {"Key", type text},
            {"Value", type text}
        }
    )
in
    #"Changed Types"
```

---

## DAX Query Patterns

### Example 1: Tag Usage Distribution
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Assignment_Tags[Key],
    "Unique_Values", DISTINCTCOUNT(Assignment_Tags[Value]),
    "Assignment_Count", DISTINCTCOUNT(Assignment_Tags[Assignment_Id]),
    "Total_Tag_Records", COUNTROWS(Assignment_Tags)
)
ORDER BY [Assignment_Count] DESC
```

### Example 2: Assignments with Specific Tag Value
```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Assignment_Tags,
        "Assignment_Id", Assignment_Tags[Assignment_Id],
        "Assignment_Name", RELATED(Assignments[Assignment_Name]),
        "Tag_Key", Assignment_Tags[Key],
        "Tag_Value", Assignment_Tags[Value],
        "Status", RELATED(Dim_Assignment_Status[Status_Name])
    ),
    Assignment_Tags[Key] = "Priority"
    && Assignment_Tags[Value] = "High"
)
ORDER BY Assignment_Tags[Assignment_Id]
```

### Example 3: All Tags for Specific Assignment
```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Assignment_Tags,
        "Assignment", RELATED(Assignments[Assignment_Name]),
        "Tag_Key", Assignment_Tags[Key],
        "Tag_Value", Assignment_Tags[Value]
    ),
    Assignment_Tags[Assignment_Id] = 12345
)
ORDER BY Assignment_Tags[Key]
```

### Example 4: Assignments with Multiple Tags
```dax
EVALUATE
FILTER(
    SUMMARIZECOLUMNS(
        Assignments[Assignment_Id],
        Assignments[Assignment_Name],
        "Tag_Count", COUNTROWS(Assignment_Tags),
        "Tag_Keys", CONCATENATEX(
            VALUES(Assignment_Tags[Key]),
            Assignment_Tags[Key],
            ", "
        )
    ),
    [Tag_Count] > 1
)
ORDER BY [Tag_Count] DESC
```

### Example 5: Tag Value Distribution for Specific Key
```dax
EVALUATE
FILTER(
    SUMMARIZECOLUMNS(
        Assignment_Tags[Value],
        FILTER(
            ALL(Assignment_Tags),
            Assignment_Tags[Key] = "Location"
        ),
        "Assignment_Count", DISTINCTCOUNT(Assignment_Tags[Assignment_Id])
    ),
    Assignment_Tags[Key] = "Location"
)
ORDER BY [Assignment_Count] DESC
```

### Example 6: Untagged Assignments
```dax
EVALUATE
SELECTCOLUMNS(
    FILTER(
        Assignments,
        NOT CONTAINS(
            Assignment_Tags,
            Assignment_Tags[Assignment_Id], Assignments[Assignment_Id]
        )
    ),
    "Assignment_Id", Assignments[Assignment_Id],
    "Assignment_Name", Assignments[Assignment_Name],
    "Status", RELATED(Dim_Assignment_Status[Status_Name]),
    "Created_Date", Assignments[Created_Date]
)
ORDER BY Assignments[Created_Date] DESC
```

---

## Data Model Pattern

### Key-Value Pair Tagging Pattern
`Assignment_Tags` implements a simple key-value pair pattern for flexible assignment metadata. Unlike the full Entity-Attribute-Value (EAV) pattern in `AssignmentPoint_Attributes`, this table uses a simplified structure without attribute grouping.

**Pattern Structure**:
- **Entity**: `Assignment_Id` identifies which assignment owns the tag
- **Key**: The tag name or category (e.g., "Priority", "Location", "Work Type")
- **Value**: The tag value assigned to that key (e.g., "High", "Pit 3", "Preventive")

**Key Characteristics**:
- **Multiple Tags per Assignment**: An assignment can have multiple tags with different keys
- **One Value per Key**: Each assignment can have only one value per key (composite primary key enforces this)
- **No Grouping**: Unlike `AssignmentPoint_Attributes`, this table doesn't include an attribute group column
- **Visibility Control**: Source filtering (IsVisible = true) allows tags to be defined but hidden from reports
- **Soft Delete**: Source filtering (IsDeleted = false) maintains referential integrity while allowing logical deletion

**Common Tag Keys**:
- **Priority**: "Low", "Medium", "High", "Critical"
- **Location**: "Pit 1", "Processing Plant", "Workshop"
- **Equipment Type**: "Haul Truck", "Excavator", "Conveyor"
- **Work Category**: "Preventive", "Corrective", "Inspection"
- **Shift**: "Day", "Night", "Swing"
- **Cost Center**: "CC-001", "CC-002"
- **Custom Tags**: Tenant-specific keys and values

**Usage in Reports**:
Tags can be used for:
1. **Dynamic Filtering**: Create slicers based on specific tag keys
2. **Conditional Formatting**: Apply formatting based on tag values (e.g., "High" priority in red)
3. **Custom Grouping**: Group assignments by tag values in visuals
4. **Search/Discovery**: Find assignments with specific tag combinations

**Example Scenario - Mining Assignment Tagging**:

Assignment ID 7001 - "Daily Haul Truck Inspection - HT-045"

Tags stored in this table:
| Key | Value |
|-----|-------|
| Priority | Medium |
| Location | Pit 3 |
| Equipment Type | Haul Truck |
| Shift | Day |
| Inspector Required | Yes |
| Safety Critical | No |

This flexible structure allows new tag keys to be added without schema changes. If a new classification is needed (e.g., "Weather Dependent"), it can be added as a new key without modifying the table structure.

**Querying Tags in DAX**:
To access a specific tag value in a measure or calculated column:
```dax
Priority_Tag = 
CALCULATE(
    MAX(Assignment_Tags[Value]),
    FILTER(
        Assignment_Tags,
        Assignment_Tags[Key] = "Priority"
        && Assignment_Tags[Assignment_Id] = EARLIER(Assignments[Assignment_Id])
    )
)
```

---

## Related Documentation
- **ERD_04_Snapshots_Details.md** - ERD diagram showing relationship context
- **Assignments.md** - Parent assignment table
- **AssignmentPoint_Attributes.md** - Similar EAV pattern for assignment point attributes
- **_Measures.md** - Centralized measures may include tag-based calculations

---

## Change History
| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-15 | Auto-generated | Initial documentation from TMDL metadata |

---

## Notes
- **Key-Value Simplicity**: This table uses a simpler structure than full EAV patterns - just key-value pairs without attribute grouping or metadata columns.
- **No Incremental Refresh**: Unlike many fact tables, this table uses standard import mode without incremental refresh, suggesting tag data volume is manageable and tag changes should be fully reflected on each refresh.
- **Composite Primary Key**: The combination of `Assignment_Id` and `Key` forms the primary key, enforcing that each assignment can have only one value per tag key.
- **Source Filtering**: The WHERE clause (IsDeleted = 0 AND IsVisible = 1) is applied at the source query level, filtering out deleted and hidden tags before import.
- **String Values Only**: All tag values are stored as strings. Numeric or date comparisons require type conversion in DAX measures.
- **Case Sensitivity**: Tag key and value comparisons in DAX are case-insensitive by default, but exact source casing is preserved.
- **Tag Key Standards**: While there are no enforced tag key standards, using consistent capitalization and naming conventions (e.g., "Priority" vs "priority" vs "PRIORITY") improves maintainability.
- **Multiple Values Pattern**: To store multiple values for the same key (e.g., multiple locations), either use comma-separated values in the Value column or append a sequence number to the Key (e.g., "Location_1", "Location_2").
- **Tag Discovery**: To discover all available tag keys, use `DISTINCT(Assignment_Tags[Key])` or create a calculated table with distinct keys.
- **No Temporal Tracking**: This table doesn't track when tags were added or modified. For historical tag analysis, consider using `Assignment_Details_Snapshot` or audit tables.
- **Performance**: Key-value lookups using CALCULATE(MAX(...), FILTER(...)) can impact performance in complex reports. Consider creating calculated columns for frequently accessed tags.
- **Tenant Isolation**: Like all tables, tags are filtered by tenant in Power Query, preventing cross-tenant tag visibility.
