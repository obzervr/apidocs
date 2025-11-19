# Fact_Entity_Table_Contents

## Overview

`Fact_Entity_Table_Contents` is a specialized fact table that stores dynamic, user-configurable entity table data within the Obzervr platform. Entity tables provide a flexible way to capture structured, repeatable data collections that are associated with work assignments, such as inspection checklists, parts lists, measurement tables, or any custom tabular data structure defined by administrators.

This table represents the intersection between assignments and their associated entity table rows, storing the actual data values captured in each cell of these dynamic tables. The table structure allows for unlimited custom entity table types without requiring schema changes to the data model.

## Columns

| Column Name | Data Type | Description |
|------------|-----------|-------------|
| **Entity_Table_Content_Id** | Integer | Primary key. Unique identifier for each entity table content record. |
| **Assignment_Id** | Integer | Foreign key to `Fact_Assignments`. Links the entity table data to a specific work assignment. |
| **Entity_Table_Id** | Integer | Foreign key to `Dim_Entity_Tables`. Identifies which entity table definition this data belongs to. |
| **Row_Id** | Integer | Identifies the row within the entity table (1, 2, 3, etc.). Multiple rows can exist per assignment/entity table combination. |
| **Column_Name** | Text | The name of the column/field within the entity table structure. |
| **Value** | Text | The actual data value stored in this cell. Stored as text regardless of data type for flexibility. |
| **FieldMeasurement_Id** | Integer | Foreign key to `Fact_FieldMeasurements`. Links to the underlying field measurement if this entity table cell is captured as a field measurement. |
| **Created_Date** | DateTime | Timestamp when this entity table content record was created. |
| **Modified_Date** | DateTime | Timestamp of the last modification to this record. |
| **Data_Type** | Text | Optional indicator of the intended data type (e.g., "Text", "Number", "Date", "Boolean") to guide type conversion. |

**Grain:** One row per entity table cell (unique combination of Assignment_Id + Entity_Table_Id + Row_Id + Column_Name).

**Key Characteristics:**
- Highly flexible EAV (Entity-Attribute-Value) pattern
- Supports dynamic table structures without schema changes
- Links to both assignments and field measurements for comprehensive tracking
- Enables user-defined repeatable data structures

## Relationships

### Active Relationships

- **To `Fact_Assignments`**: Many-to-one relationship on `Assignment_Id` field
  - Connects entity table data to the parent work assignment
  - Enables filtering entity data by assignment status, date, location, etc.
  - Subject to assignment-level security and filtering
  
- **To `Dim_Entity_Tables`**: Many-to-one relationship on `Entity_Table_Id` field
  - Links to the entity table definition (structure, columns, purpose)
  - Enables grouping and filtering by entity table type

### Inactive Relationships

- **To `Fact_FieldMeasurements`**: Many-to-one relationship on `FieldMeasurement_Id` field
  - Often inactive to avoid ambiguity with direct Assignment_Id relationship
  - Use USERELATIONSHIP() to activate when needed for field measurement analysis
  - Enables tracing entity table cells back to source field measurements

## Data Source

**Type:** SQL Database Table

**Source System:** Obzervr application database (Entity Table module)

**Refresh Type:** Import or DirectQuery depending on configuration and data volume

**Typical SQL Structure:**
```sql
SELECT 
    Entity_Table_Content_Id,
    Assignment_Id,
    Entity_Table_Id,
    Row_Id,
    Column_Name,
    Value,
    FieldMeasurement_Id,
    Created_Date,
    Modified_Date,
    Data_Type
FROM dbo.EntityTableContents
ORDER BY Assignment_Id, Entity_Table_Id, Row_Id, Column_Name
```

## DAX Patterns

### Pivot Entity Table Data (Table Visual)
```dax
Entity Table Pivot = 
CALCULATETABLE(
    SUMMARIZE(
        Fact_Entity_Table_Contents,
        Fact_Entity_Table_Contents[Row_Id],
        Fact_Entity_Table_Contents[Column_Name],
        "Value", MAX(Fact_Entity_Table_Contents[Value])
    ),
    ALLEXCEPT(Fact_Entity_Table_Contents, Fact_Entity_Table_Contents[Assignment_Id])
)
```

### Count Entity Table Rows per Assignment
```dax
Entity Table Row Count = 
CALCULATE(
    DISTINCTCOUNT(Fact_Entity_Table_Contents[Row_Id])
)
```

### Get Specific Column Value
```dax
Part Number = 
CALCULATE(
    MAX(Fact_Entity_Table_Contents[Value]),
    Fact_Entity_Table_Contents[Column_Name] = "Part_Number"
)
```

### Sum Numeric Column (with type conversion)
```dax
Total Quantity = 
SUMX(
    FILTER(
        Fact_Entity_Table_Contents,
        Fact_Entity_Table_Contents[Column_Name] = "Quantity"
    ),
    VALUE(Fact_Entity_Table_Contents[Value])
)
```

### Entity Table Completion Check
```dax
Is Entity Table Complete = 
VAR RequiredCells = 
    CALCULATE(
        COUNTROWS(Dim_Entity_Table_Columns),
        RELATED(Dim_Entity_Tables[Entity_Table_Id]) = [Selected Entity Table]
    )
VAR CompletedCells = 
    CALCULATE(
        COUNTROWS(Fact_Entity_Table_Contents),
        Fact_Entity_Table_Contents[Value] <> BLANK()
    )
RETURN
    CompletedCells >= RequiredCells
```

### Get Latest Modified Row
```dax
Last Modified Row = 
CALCULATE(
    MAX(Fact_Entity_Table_Contents[Row_Id]),
    TOPN(
        1,
        VALUES(Fact_Entity_Table_Contents[Row_Id]),
        MAX(Fact_Entity_Table_Contents[Modified_Date]),
        DESC
    )
)
```

### Dynamic Column Aggregation
```dax
Dynamic Column Sum = 
VAR SelectedColumn = SELECTEDVALUE(Parameters[Column_Name])
RETURN
    SUMX(
        FILTER(
            Fact_Entity_Table_Contents,
            Fact_Entity_Table_Contents[Column_Name] = SelectedColumn
        ),
        VALUE(Fact_Entity_Table_Contents[Value])
    )
```

## Common Usage Scenarios

1. **Inspection Checklists**
   - Display multi-row inspection results
   - Calculate completion percentages
   - Aggregate pass/fail counts

2. **Parts Lists**
   - Show parts used per assignment
   - Calculate total quantities and costs
   - Track part number usage

3. **Measurement Tables**
   - Display time-series measurements in tabular format
   - Calculate min/max/average across rows
   - Identify out-of-spec measurements

4. **Custom Data Grids**
   - Render user-defined data structures
   - Support dynamic column sets
   - Enable flexible data capture patterns

5. **Entity Table Analytics**
   - Analyze most common values per column
   - Track data entry patterns
   - Calculate average rows per assignment

## Related Tables

- **Fact_Assignments**: Parent assignment for entity table data
- **Dim_Entity_Tables**: Entity table definitions and metadata
- **Dim_Entity_Table_Columns**: Column definitions for entity tables
- **Fact_FieldMeasurements**: Source field measurements that populate entity tables
- **Dim_WorkTemplates**: Templates that include entity table configurations
- **Dim_Field_Fragments**: Field fragments configured to use entity table capture

## EAV Pattern Considerations

**Advantages:**
- Extreme flexibility for dynamic table structures
- No schema changes required for new entity table types
- Supports variable column counts per entity table

**Challenges:**
- Queries can be complex for pivoting data to columnar format
- Type conversion required for numeric/date aggregations
- Performance considerations for large entity tables
- Less intuitive than traditional relational structure

**Best Practices:**
- Use calculated tables to pivot frequently-accessed entity tables into columnar format
- Create specific measures for common column aggregations
- Include Data_Type column to guide type conversion
- Consider column-based storage format (import mode) for better compression

## Performance Considerations

- High cardinality due to EAV structure (many rows per logical entity table)
- Index on Assignment_Id for efficient assignment-based filtering
- Index on Entity_Table_Id + Row_Id for row-level queries
- Consider aggregated or pivoted tables for frequently-accessed entity table reports
- DirectQuery may be necessary for very large entity table datasets

## Change History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-01-23 | Initial documentation |
