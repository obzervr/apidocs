# Assignment_FieldMeasurement_Exceptions

## Table Overview
`Assignment_FieldMeasurement_Exceptions` is a fact table that tracks exceptions and anomalies detected in field measurement readings. Each row represents a single exception raised for a specific field measurement, capturing who raised it, when, resolution details, and priority classification.

This table enables tracking of out-of-range readings, data quality issues, equipment anomalies, and their resolution workflow.

**Current Status**: Standard import table (no incremental refresh specified in TMDL).

---

## Specifications
- **Source**: `VW_AssignmentExceptions` view
- **Row Count**: Variable (depends on exception frequency and resolution rate)
- **Grain**: One row per field measurement exception
- **Primary Key**: `Id` (deduplicated in Power Query)
- **Incremental Refresh**: Not enabled
- **Partitioning Strategy**: Standard import
- **Source Columns**: 15
- **Calculated Columns**: 1 (Is_Resolved)
- **Date Keys**: 2 (Raised_At_DateKey, Resolved_At_DateKey)

---

## Column Specifications

| Column Name | Data Type | Format | Nullable | Hidden | Description |
|------------|-----------|--------|----------|--------|-------------|
| Id | Int64 | | No | No | Primary key for the exception record |
| Tenant_Id | Int64 | | No | No | Foreign key to tenant dimension |
| Assignment_Id | Int64 | | No | No | Foreign key to the assignment containing the exception |
| FieldMeasurement_Id | Int64 | | No | No | Foreign key to the specific field measurement with the exception |
| Priority | Int64 | | No | No | Exception priority level (lower numbers = higher priority) |
| Raised_At | DateTime | yyyy-MM-dd HH:mm:ss | No | No | Timestamp when the exception was raised |
| Raised_By | Int64 | | Yes | No | User ID of the person who raised the exception |
| Raised_By_Name | String | | Yes | No | Full name of the person who raised the exception |
| Resolved_At | DateTime | yyyy-MM-dd HH:mm:ss | Yes | No | Timestamp when the exception was resolved (NULL if unresolved) |
| Resolved_By | Int64 | | Yes | No | User ID of the person who resolved the exception |
| Resolved_By_Name | String | | Yes | No | Full name of the person who resolved the exception |
| Comment | String | | Yes | No | User-entered comments about the exception |
| Last_Updated | DateTime | yyyy-MM-dd HH:mm:ss | Yes | No | Timestamp of last modification |
| Created_Date | DateTime | yyyy-MM-dd HH:mm:ss | No | No | Creation timestamp |
| Raised_At_DateKey | Int64 | YYYYMMDD | Yes | No | Date key derived from Raised_At timestamp |
| Resolved_At_DateKey | Int64 | YYYYMMDD | Yes | No | Date key derived from Resolved_At timestamp (NULL if unresolved) |

---

## Calculated Columns

### Is_Resolved
Binary indicator ("0" or "1") showing whether the exception has been resolved. Based on the presence of a `Resolved_At` timestamp.

```dax
Is_Resolved = 
IF(
    ISBLANK(Assignment_FieldMeasurement_Exceptions[Resolved_At]),
    "0",
    "1"
)
```

---

## Relationships

### Outbound Relationships
| To Table | From Column(s) | To Column(s) | Cardinality | Cross Filter | Relationship ID |
|----------|---------------|--------------|-------------|--------------|-----------------|
| `Dim_Tenant` | Tenant_Id | Tenant_Id | Many-to-One | Single | (Tenant context) |
| `Assignments` | Assignment_Id | Assignment_Id | Many-to-One | Single | (Assignment context) |
| `TimeSeries_FieldMeasurements` | FieldMeasurement_Id | Id | Many-to-One | Single | (Field measurement context) |
| `Dim_Date_Reference` | Raised_At_DateKey | Date_Key | Many-to-One | Single | (Raised date - role-playing) |
| `Dim_Date_Reference` | Resolved_At_DateKey | Date_Key | Many-to-One | Single | (Resolved date - role-playing) |

### Inbound Relationships
None. This is a leaf table in the relationship hierarchy.

---

## Power Query M Source

```m
let
    Source = Value.NativeQuery(
        Obzervr_DataWarehouse,
        "SELECT 
            Id,
            Tenant_Id,
            Assignment_Id,
            FieldMeasurement_Id,
            Priority,
            Raised_At,
            Raised_By,
            Raised_By_Name,
            Resolved_At,
            Resolved_By,
            Resolved_By_Name,
            Comment,
            Last_Updated,
            Created_Date
        FROM [dbo].[VW_AssignmentExceptions]"
    ),
    #"Filtered Tenant" = Table.SelectRows(
        Source,
        each [Tenant_Id] = TenantId
    ),
    #"Deduplicated by Id" = Table.Distinct(
        #"Filtered Tenant",
        {"Id"}
    ),
    #"Changed Types" = Table.TransformColumnTypes(
        #"Deduplicated by Id",
        {
            {"Id", Int64.Type},
            {"Tenant_Id", Int64.Type},
            {"Assignment_Id", Int64.Type},
            {"FieldMeasurement_Id", Int64.Type},
            {"Priority", Int64.Type},
            {"Raised_At", type datetime},
            {"Raised_By", Int64.Type},
            {"Raised_By_Name", type text},
            {"Resolved_At", type datetime},
            {"Resolved_By", Int64.Type},
            {"Resolved_By_Name", type text},
            {"Comment", type text},
            {"Last_Updated", type datetime},
            {"Created_Date", type datetime}
        }
    ),
    #"Uppercased Text Columns" = Table.TransformColumns(
        #"Changed Types",
        {
            {"Raised_By_Name", Text.Upper},
            {"Resolved_By_Name", Text.Upper},
            {"Comment", Text.Upper}
        }
    ),
    #"Added Raised_At_DateKey" = Table.AddColumn(
        #"Uppercased Text Columns",
        "Raised_At_DateKey",
        each Date.Year([Raised_At]) * 10000 
            + Date.Month([Raised_At]) * 100 
            + Date.Day([Raised_At]),
        Int64.Type
    ),
    #"Added Resolved_At_DateKey" = Table.AddColumn(
        #"Added Raised_At_DateKey",
        "Resolved_At_DateKey",
        each if [Resolved_At] = null then null
            else Date.Year([Resolved_At]) * 10000 
                + Date.Month([Resolved_At]) * 100 
                + Date.Day([Resolved_At]),
        Int64.Type
    )
in
    #"Added Resolved_At_DateKey"
```

---

## DAX Query Patterns

### Example 1: Open Exceptions by Priority
```dax
EVALUATE
FILTER(
    SUMMARIZECOLUMNS(
        Assignment_FieldMeasurement_Exceptions[Priority],
        FILTER(
            ALL(Assignment_FieldMeasurement_Exceptions),
            Assignment_FieldMeasurement_Exceptions[Is_Resolved] = "0"
        ),
        "Open_Exception_Count", COUNTROWS(Assignment_FieldMeasurement_Exceptions),
        "Unique_Assignments", DISTINCTCOUNT(Assignment_FieldMeasurement_Exceptions[Assignment_Id]),
        "Unique_Field_Measurements", DISTINCTCOUNT(Assignment_FieldMeasurement_Exceptions[FieldMeasurement_Id]),
        "Oldest_Exception", MIN(Assignment_FieldMeasurement_Exceptions[Raised_At])
    ),
    Assignment_FieldMeasurement_Exceptions[Is_Resolved] = "0"
)
ORDER BY Assignment_FieldMeasurement_Exceptions[Priority]
```

### Example 2: Exception Resolution Time Analysis
```dax
EVALUATE
ADDCOLUMNS(
    FILTER(
        Assignment_FieldMeasurement_Exceptions,
        Assignment_FieldMeasurement_Exceptions[Is_Resolved] = "1"
    ),
    "Assignment_Name", RELATED(Assignments[Assignment_Name]),
    "Field_Measurement", RELATED(TimeSeries_FieldMeasurements[FieldMeasurement_Name]),
    "Resolution_Time_Hours", DATEDIFF(
        Assignment_FieldMeasurement_Exceptions[Raised_At],
        Assignment_FieldMeasurement_Exceptions[Resolved_At],
        HOUR
    ),
    "Raised_By", Assignment_FieldMeasurement_Exceptions[Raised_By_Name],
    "Resolved_By", Assignment_FieldMeasurement_Exceptions[Resolved_By_Name]
)
ORDER BY Assignment_FieldMeasurement_Exceptions[Priority], [Resolution_Time_Hours] DESC
```

### Example 3: Exception Trend by Month
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_Date_Reference[Calendar_Year],
    Dim_Date_Reference[Calendar_Month_Name],
    "Total_Raised", COUNTROWS(Assignment_FieldMeasurement_Exceptions),
    "Total_Resolved", COUNTROWS(
        FILTER(
            Assignment_FieldMeasurement_Exceptions,
            Assignment_FieldMeasurement_Exceptions[Is_Resolved] = "1"
        )
    ),
    "Still_Open", COUNTROWS(
        FILTER(
            Assignment_FieldMeasurement_Exceptions,
            Assignment_FieldMeasurement_Exceptions[Is_Resolved] = "0"
        )
    ),
    "Avg_Resolution_Days", AVERAGE(
        FILTER(
            Assignment_FieldMeasurement_Exceptions,
            Assignment_FieldMeasurement_Exceptions[Is_Resolved] = "1"
        ),
        DATEDIFF(
            Assignment_FieldMeasurement_Exceptions[Raised_At],
            Assignment_FieldMeasurement_Exceptions[Resolved_At],
            DAY
        )
    )
)
ORDER BY Dim_Date_Reference[Calendar_Year], Dim_Date_Reference[Month_Number]
```

### Example 4: Top Exception Raisers
```dax
EVALUATE
TOPN(
    20,
    SUMMARIZECOLUMNS(
        Assignment_FieldMeasurement_Exceptions[Raised_By_Name],
        "Total_Exceptions_Raised", COUNTROWS(Assignment_FieldMeasurement_Exceptions),
        "High_Priority_Count", COUNTROWS(
            FILTER(
                Assignment_FieldMeasurement_Exceptions,
                Assignment_FieldMeasurement_Exceptions[Priority] = 1
            )
        ),
        "Avg_Priority", AVERAGE(Assignment_FieldMeasurement_Exceptions[Priority]),
        "Most_Recent_Exception", MAX(Assignment_FieldMeasurement_Exceptions[Raised_At])
    ),
    [Total_Exceptions_Raised],
    DESC
)
```

### Example 5: Unresolved Exceptions Report
```dax
EVALUATE
TOPN(
    100,
    SELECTCOLUMNS(
        FILTER(
            Assignment_FieldMeasurement_Exceptions,
            Assignment_FieldMeasurement_Exceptions[Is_Resolved] = "0"
        ),
        "Exception_Id", Assignment_FieldMeasurement_Exceptions[Id],
        "Priority", Assignment_FieldMeasurement_Exceptions[Priority],
        "Assignment", RELATED(Assignments[Assignment_Name]),
        "Field_Measurement", RELATED(TimeSeries_FieldMeasurements[FieldMeasurement_Name]),
        "Raised_At", Assignment_FieldMeasurement_Exceptions[Raised_At],
        "Raised_By", Assignment_FieldMeasurement_Exceptions[Raised_By_Name],
        "Days_Open", DATEDIFF(
            Assignment_FieldMeasurement_Exceptions[Raised_At],
            TODAY(),
            DAY
        ),
        "Comment", Assignment_FieldMeasurement_Exceptions[Comment]
    ),
    [Days_Open],
    DESC
)
```

### Example 6: Exception Resolution Performance by User
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Assignment_FieldMeasurement_Exceptions[Resolved_By_Name],
    FILTER(
        ALL(Assignment_FieldMeasurement_Exceptions),
        Assignment_FieldMeasurement_Exceptions[Is_Resolved] = "1"
    ),
    "Total_Resolved", COUNTROWS(Assignment_FieldMeasurement_Exceptions),
    "Avg_Resolution_Hours", AVERAGE(
        DATEDIFF(
            Assignment_FieldMeasurement_Exceptions[Raised_At],
            Assignment_FieldMeasurement_Exceptions[Resolved_At],
            HOUR
        )
    ),
    "High_Priority_Resolved", COUNTROWS(
        FILTER(
            Assignment_FieldMeasurement_Exceptions,
            Assignment_FieldMeasurement_Exceptions[Priority] = 1
        )
    ),
    "Last_Resolution_Date", MAX(Assignment_FieldMeasurement_Exceptions[Resolved_At])
)
ORDER BY [Total_Resolved] DESC
```

---

## Data Model Pattern

### Exception Tracking Workflow Pattern
`Assignment_FieldMeasurement_Exceptions` implements a two-state workflow for tracking data quality and measurement anomalies:

**Exception Lifecycle**:
1. **Raised**: Exception created when anomaly detected
   - `Raised_At` timestamp captured
   - `Raised_By` user recorded
   - `Priority` assigned (lower number = higher priority)
   - `Comment` describes the issue

2. **Resolved**: Exception addressed and closed
   - `Resolved_At` timestamp captured
   - `Resolved_By` user recorded
   - `Comment` may include resolution notes
   - `Is_Resolved` calculated column becomes "1"

**Priority Levels**:
The `Priority` column uses integer values where lower numbers indicate higher priority. Common patterns:
- **1** = Critical (immediate action required)
- **2** = High (urgent attention needed)
- **3** = Medium (address within standard timeframe)
- **4+** = Low (backlog/informational)

**Exception Sources**:
Exceptions are typically triggered by:
- **Boundary Violations**: Numeric readings outside `Lower_Boundary` / `Upper_Boundary` ranges in `TimeSeries_FieldMeasurements`
- **Data Quality Issues**: Missing required measurements, invalid data types
- **Equipment Anomalies**: Unexpected sensor readings indicating potential equipment failure
- **Manual Identification**: Field technicians or supervisors flagging concerns

**Relationship to Field Measurements**:
Each exception links to a specific field measurement (`FieldMeasurement_Id` → `TimeSeries_FieldMeasurements.Id`). This enables:
- Viewing the actual reading value that triggered the exception
- Understanding the measurement context (timeseries, section, boundaries)
- Tracking exception patterns for specific field measurement types

**Resolution Tracking**:
The NULL/NOT NULL state of `Resolved_At` determines exception status:
- **NULL**: Exception is open/unresolved
- **NOT NULL**: Exception has been resolved

The calculated column `Is_Resolved` provides a binary indicator ("0"/"1") for easier filtering in reports and measures.

**Date Key Relationships**:
Two date keys enable separate temporal analysis:
- `Raised_At_DateKey`: When exceptions were raised (exception creation trends)
- `Resolved_At_DateKey`: When exceptions were resolved (resolution performance)

These separate relationships to `Dim_Date_Reference` enable analysis like "exceptions raised in Q1 vs resolved in Q1" (which may differ if exceptions cross period boundaries).

**Example Scenario - Out-of-Range Temperature Reading**:

Field Measurement ID 5001 - "Engine Coolant Temperature" reading "105°C" (Upper_Boundary = 95°C)

Exception Record:
- **Id**: 789
- **Priority**: 2 (High - potential engine damage)
- **Raised_At**: 2024-11-15 14:30:00
- **Raised_By**: System (automated boundary check)
- **Comment**: "COOLANT TEMP EXCEEDS SAFE LIMIT. ENGINE SHUTDOWN REQUIRED."
- **Resolved_At**: 2024-11-15 16:45:00 (2.25 hours later)
- **Resolved_By**: Maintenance technician ID 42
- **Resolved_By_Name**: "JOHN SMITH"
- **Comment** (updated): "COOLANT SYSTEM LEAK REPAIRED. SYSTEM TESTED AND WITHIN NORMAL RANGE."

This exception demonstrates:
- Automated detection (raised by system)
- High priority assignment (potential damage)
- Resolution workflow (technician resolved)
- Time tracking (2.25-hour resolution time)

---

## Related Documentation
- **ERD_04_Snapshots_Details.md** - ERD diagram showing relationship context
- **TimeSeries_FieldMeasurements.md** - Source of field measurements that may trigger exceptions
- **Assignments.md** - Assignment context for exceptions
- **Dim_Date_Reference.md** - Date dimension for temporal analysis with role-playing relationships
- **_Measures.md** - Centralized measures including exception-specific calculations

---

## Change History
| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-15 | Auto-generated | Initial documentation from TMDL metadata |

---

## Notes
- **Two-State Workflow**: Exceptions have two states - open (Resolved_At = NULL) and resolved (Resolved_At = NOT NULL). There is no "In Progress" or intermediate state.
- **Priority Numbering**: Lower priority numbers indicate higher priority (1 = highest). This pattern is common in IT systems but may be counterintuitive.
- **Deduplication**: The Power Query includes `Table.Distinct` on the `Id` column, suggesting the source view may occasionally return duplicate records.
- **Text Uppercasing**: User names and comments are uppercased in Power Query (`Text.Upper`), providing consistent casing for sorting and display.
- **Date Key Calculation**: Both date keys use the YYYYMMDD formula in Power Query, calculated at import rather than in DAX for performance.
- **NULL Resolved_At**: Unresolved exceptions have NULL in `Resolved_At`, `Resolved_By`, `Resolved_By_Name`, and `Resolved_At_DateKey`.
- **Is_Resolved String**: The calculated column uses "0"/"1" strings rather than TRUE/FALSE booleans, likely for compatibility with source system conventions or slicer display.
- **Raised_By vs Raised_By_Name**: Both the user ID and full name are stored, providing flexibility for joins and display without requiring user dimension lookups.
- **Comment Updates**: The `Comment` column may be updated during resolution. The `Last_Updated` timestamp tracks when any column was last modified.
- **No Incremental Refresh**: Unlike many fact tables, this table doesn't use incremental refresh, suggesting exception volume is manageable for full refresh or exceptions need to be re-evaluated on each refresh cycle.
- **Role-Playing Dates**: The two date key relationships to `Dim_Date_Reference` are role-playing relationships, requiring explicit relationship specification in DAX measures (e.g., `USERELATIONSHIP`).
- **Tenant Filtering**: Standard tenant filtering applied in Power Query ensures cross-tenant exception isolation.
- **Priority Analysis**: Lower average priority values indicate more critical exception profiles for assignments, field measurements, or users.
- **Resolution Time Calculation**: Calculate using `DATEDIFF(Raised_At, Resolved_At, HOUR/DAY)` for performance metrics.
