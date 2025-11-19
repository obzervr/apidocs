# Fact_Lookup_Tables

## Overview

`Fact_Lookup_Tables` is a fact table that stores the actual data values and entries for TSFM (Template, Section, Fragment, Measurement) lookup tables. This table works in conjunction with `Dim_Lookup_Tables` to provide a flexible, user-configurable lookup list system within the Obzervr platform.

Lookup tables enable dynamic dropdown lists and selection options for field fragments, allowing administrators to define and maintain reference data (e.g., equipment types, priority levels, status codes) without requiring data model changes. Each row in this table represents a single entry/option within a specific lookup table.

## Columns

| Column Name | Data Type | Description |
|------------|-----------|-------------|
| **Lookup_Value_Id** | Integer | Primary key. Unique identifier for each lookup table entry. |
| **Table_Id** | Integer | Foreign key to `Dim_Lookup_Tables`. Identifies which lookup table this entry belongs to. |
| **Value** | Text | The actual lookup value/option that will be displayed and stored when selected. |
| **Display_Order** | Integer | Numeric sort order for controlling the sequence of options in dropdown lists. Lower numbers appear first. |
| **Is_Active** | Boolean | Flag indicating whether this lookup value is currently active and should be displayed as an option. |
| **Created_Date** | DateTime | Timestamp when this lookup entry was created. |
| **Modified_Date** | DateTime | Timestamp of the last modification to this entry. |
| **Description** | Text | Optional detailed description or notes about this lookup value. |

**Grain:** One row per lookup table entry/option.

**Key Characteristics:**
- Supports dynamic, user-configurable reference data
- Enables sorting via Display_Order for UX control
- Soft delete pattern via Is_Active flag (preserves historical data)
- Generic design allows unlimited lookup table types

## Relationships

### Active Relationships

- **To `Dim_Lookup_Tables`**: Many-to-one relationship on `Table_Id` field
  - Connects each lookup value to its parent lookup table definition
  - Enables filtering by lookup table name/type
  - Supports cascading filters from table to values

### Inactive Relationships

- May have inactive relationships to other fact tables that use lookup values as foreign keys
- Role-playing dimension pattern if lookup values are referenced from multiple contexts

## Data Source

**Type:** SQL Database Table

**Source System:** Obzervr application database (TSFM module)

**Refresh Type:** Import or DirectQuery depending on configuration

**Typical SQL Structure:**
```sql
SELECT 
    Lookup_Value_Id,
    Table_Id,
    Value,
    Display_Order,
    Is_Active,
    Created_Date,
    Modified_Date,
    Description
FROM dbo.LookupTableValues
WHERE Is_Active = 1  -- Optional filter to only import active values
ORDER BY Table_Id, Display_Order
```

## DAX Patterns

### Get Active Lookup Values
```dax
Active Lookup Values = 
CALCULATE(
    COUNTROWS(Fact_Lookup_Tables),
    Fact_Lookup_Tables[Is_Active] = TRUE()
)
```

### Lookup Value Display List (for specific table)
```dax
Priority Levels = 
CALCULATE(
    VALUES(Fact_Lookup_Tables[Value]),
    Dim_Lookup_Tables[Table_Name] = "Priority Levels",
    Fact_Lookup_Tables[Is_Active] = TRUE()
)
```

### Get Sorted Lookup Values
```dax
Sorted Lookup Options = 
CALCULATETABLE(
    Fact_Lookup_Tables,
    Fact_Lookup_Tables[Is_Active] = TRUE(),
    Dim_Lookup_Tables[Table_Name] = "Equipment Types"
)
// Then use Display_Order for sorting in visual
```

### Check If Value Exists in Lookup
```dax
Is Valid Lookup Value = 
VAR SelectedValue = SELECTEDVALUE(FieldMeasurements[Value])
VAR IsValid = 
    CALCULATE(
        COUNTROWS(Fact_Lookup_Tables),
        Fact_Lookup_Tables[Value] = SelectedValue,
        Fact_Lookup_Tables[Is_Active] = TRUE(),
        Dim_Lookup_Tables[Table_Name] = "Valid Options"
    ) > 0
RETURN
    IsValid
```

### Count Lookup Options by Table
```dax
Lookup Options Count = 
CALCULATE(
    COUNTROWS(Fact_Lookup_Tables),
    Fact_Lookup_Tables[Is_Active] = TRUE()
)
```

### Most Recently Modified Lookup Entry
```dax
Last Modified Date = 
MAX(Fact_Lookup_Tables[Modified_Date])
```

## Common Usage Scenarios

1. **Dropdown List Population**
   - Provide dynamic options for field fragment selection
   - Filter by Table_Id to get specific lookup list
   - Sort by Display_Order for proper UX sequence

2. **Lookup Value Validation**
   - Verify submitted values exist in active lookup tables
   - Check referential integrity for field measurements
   - Identify invalid or deprecated value usage

3. **Lookup Table Administration**
   - Display lookup table contents for editing
   - Show creation/modification audit trail
   - Track active vs inactive options

4. **Reference Data Analytics**
   - Count distinct values per lookup table
   - Analyze lookup value usage frequency
   - Identify unused lookup options

5. **Data Quality Monitoring**
   - Find orphaned lookup values (no matching Table_Id)
   - Identify lookup tables with no active values
   - Track lookup table growth over time

## Related Tables

- **Dim_Lookup_Tables**: Parent table defining lookup table metadata (name, description, type)
- **Dim_Published_Field_Fragments**: Field fragments that reference lookup tables for dropdown options
- **Fact_FieldMeasurements**: Actual field measurements that may store selected lookup values
- **Dim_Field_Types**: Field type definitions that specify when lookup tables should be used

## Data Integrity Notes

- Lookup values are typically soft-deleted (Is_Active = FALSE) rather than physically removed to preserve historical data integrity
- Display_Order should be unique within each Table_Id for consistent sorting
- Value field should be unique within each Table_Id to avoid ambiguity
- Foreign key constraint on Table_Id ensures all lookup values belong to defined lookup tables

## Performance Considerations

- Index on Table_Id for efficient filtering by lookup table
- Index on Is_Active for quick active value filtering
- Consider separate indexes on Value if used for text searching
- Small table size typically allows for full import mode

## Change History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-01-23 | Initial documentation |
