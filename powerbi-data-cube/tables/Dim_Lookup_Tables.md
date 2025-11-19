# Dim_Lookup_Tables

## Overview

`Dim_Lookup_Tables` is a dimension table that defines the metadata and structure for user-configurable lookup tables within the Obzervr platform. This table works in conjunction with `Fact_Lookup_Tables` to provide a flexible reference data management system.

Each row in this dimension represents a distinct lookup table definition (e.g., "Equipment Types", "Priority Levels", "Status Codes"), which can then have multiple lookup values/entries stored in the related fact table. This design pattern enables administrators to create and maintain custom dropdown lists and selection options without requiring database schema changes.

## Columns

| Column Name | Data Type | Description |
|------------|-----------|-------------|
| **Table_Id** | Integer | Primary key. Unique identifier for each lookup table definition. |
| **Table_Name** | Text | The name of the lookup table. Used for reference in reports and as display label. |
| **Description** | Text | Detailed description of the lookup table's purpose and usage. |
| **Table_Type** | Text | Optional categorization of the lookup table (e.g., "System", "User-Defined", "Template"). |
| **Is_Active** | Boolean | Flag indicating whether this lookup table is currently active and available for use. |
| **Created_Date** | DateTime | Timestamp when this lookup table definition was created. |
| **Modified_Date** | DateTime | Timestamp of the last modification to this lookup table definition. |
| **Created_By** | Text | User who created this lookup table definition. |
| **Module** | Text | Optional field indicating which Obzervr module or context uses this lookup table (e.g., "TSFM", "Maintenance", "Safety"). |
| **Allow_Multiple_Selection** | Boolean | Flag indicating whether this lookup supports multi-select behavior in the UI. |

**Grain:** One row per lookup table definition.

**Key Characteristics:**
- Central registry for all lookup table definitions
- Supports categorization via Table_Type and Module
- Audit trail via Created_Date, Modified_Date, Created_By
- Enables both single-select and multi-select lookup behaviors

## Relationships

### Active Relationships

- **From `Fact_Lookup_Tables`**: One-to-many relationship
  - Each lookup table definition can have multiple lookup values/entries
  - Enables filtering lookup values by table name or type
  - Primary navigation path: Dim_Lookup_Tables → Fact_Lookup_Tables

### Inactive Relationships

- May have inactive relationships to other dimension tables that reference lookup tables
- Could have role-playing relationships if lookup tables are used in multiple contexts

## Data Source

**Type:** SQL Database Table

**Source System:** Obzervr application database (TSFM configuration)

**Refresh Type:** Import (typically small, relatively static reference data)

**Typical SQL Structure:**
```sql
SELECT 
    Table_Id,
    Table_Name,
    Description,
    Table_Type,
    Is_Active,
    Created_Date,
    Modified_Date,
    Created_By,
    Module,
    Allow_Multiple_Selection
FROM dbo.LookupTableDefinitions
WHERE Is_Active = 1
ORDER BY Table_Name
```

## DAX Patterns

### Get Active Lookup Tables
```dax
Active Lookup Tables = 
CALCULATE(
    COUNTROWS(Dim_Lookup_Tables),
    Dim_Lookup_Tables[Is_Active] = TRUE()
)
```

### Lookup Table Dropdown (for parameter selection)
```dax
Lookup Table Names = 
CALCULATETABLE(
    VALUES(Dim_Lookup_Tables[Table_Name]),
    Dim_Lookup_Tables[Is_Active] = TRUE()
)
```

### Count Lookup Values per Table
```dax
Lookup Value Count = 
CALCULATE(
    COUNTROWS(Fact_Lookup_Tables),
    Fact_Lookup_Tables[Is_Active] = TRUE()
)
```

### Filter by Module
```dax
TSFM Lookup Tables = 
CALCULATE(
    COUNTROWS(Dim_Lookup_Tables),
    Dim_Lookup_Tables[Module] = "TSFM",
    Dim_Lookup_Tables[Is_Active] = TRUE()
)
```

### Get Lookup Table Description
```dax
Selected Lookup Description = 
SELECTEDVALUE(Dim_Lookup_Tables[Description], "No lookup table selected")
```

### Check if Multi-Select Enabled
```dax
Supports Multi-Select = 
SELECTEDVALUE(Dim_Lookup_Tables[Allow_Multiple_Selection], FALSE())
```

### Recently Modified Lookup Tables
```dax
Recently Modified = 
CALCULATE(
    COUNTROWS(Dim_Lookup_Tables),
    Dim_Lookup_Tables[Modified_Date] >= TODAY() - 30
)
```

## Common Usage Scenarios

1. **Lookup Table Administration**
   - List all available lookup tables
   - Display lookup table metadata and descriptions
   - Show creation/modification audit trail
   - Enable/disable lookup tables

2. **Lookup Table Selection Parameters**
   - Provide dropdown for report users to select lookup tables
   - Filter related lookup values based on selected table
   - Dynamic report configuration

3. **Configuration Management**
   - Track which modules use which lookup tables
   - Identify system vs user-defined lookup tables
   - Monitor lookup table usage patterns

4. **Lookup Table Analytics**
   - Count total lookup tables by type/module
   - Analyze lookup table growth over time
   - Identify unused or under-utilized lookup tables

5. **Data Governance**
   - Show who created/modified lookup tables
   - Track lookup table lifecycle
   - Support lookup table retirement workflow

## Related Tables

- **Fact_Lookup_Tables**: Contains the actual lookup values/entries for each lookup table
- **Dim_Published_Field_Fragments**: Field fragments that reference lookup tables for dropdown options
- **Fact_FieldMeasurements**: Field measurements that store selected lookup values
- **Dim_Field_Types**: Field type definitions that specify when lookup tables should be used

## Dimension Characteristics

**Type:** Type 1 Slowly Changing Dimension
- Updates overwrite existing values
- Modified_Date tracks when changes occurred
- No historical versioning of lookup table definitions

**Hierarchy Potential:**
- Module → Table_Type → Table_Name (natural hierarchy for navigation)
- Can support drill-down from high-level groupings to specific lookup tables

**Attributes for Slicing:**
- Table_Type (system vs user-defined)
- Module (functional area)
- Is_Active (active vs inactive)
- Allow_Multiple_Selection (single vs multi-select behavior)

## Data Integrity Notes

- Table_Name should be unique for clarity and to avoid ambiguity
- Soft-delete pattern via Is_Active flag preserves historical references
- Foreign key constraint ensures Fact_Lookup_Tables entries reference valid lookup tables
- Created_By should reference a valid user in the system

## Performance Considerations

- Small dimension table (typically <100 rows)
- Full import mode recommended for optimal performance
- No special indexing typically required
- Consider caching calculated columns if complex transformations are needed

## Change History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-01-23 | Initial documentation |
