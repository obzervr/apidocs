# TimeSeries_FieldMeasurements

## Table Overview
`TimeSeries_FieldMeasurements` is an incremental refresh fact table that stores individual field measurement readings captured within timeseries instances. Each row represents a single field measurement taken as part of a work assignment's data collection process. The table includes reading values, boundaries, timestamps, and data type classification to enable type-specific analysis.

This table has a many-to-one relationship with `TimeSeries` (multiple field measurements per timeseries instance) and serves as the detailed measurement layer beneath the timeseries container structure.

**Current Status**: Incremental Refresh enabled with 5-year rolling window and 1-year increments based on `Created_Date`.

---

## Specifications
- **Source**: `VW_TimeSeriesFieldMeasurements` view
- **Row Count**: High volume (field measurements for all assignments across 5-year window)
- **Grain**: One row per field measurement within a timeseries instance
- **Primary Key**: `Id`
- **Incremental Refresh**: 5 years rolling, 1-year increments, filtered on `Created_Date`
- **Partitioning Strategy**: Incremental refresh by created date
- **Source Columns**: 25
- **Calculated Columns**: 2 (hour columns with 15-minute rounding)
- **Date Keys**: 2 (Captured_On_Datekey, Completed_At_Datekey)

---

## Column Specifications

| Column Name | Data Type | Format | Nullable | Hidden | Description |
|------------|-----------|--------|----------|--------|-------------|
| Tenant_Id | Int64 | | No | No | Foreign key to tenant dimension |
| Id | Int64 | | No | No | Primary key for field measurement |
| TimeSeries_Id | Int64 | | No | No | Foreign key to parent timeseries instance |
| FieldMeasurement_Name | String | | Yes | No | Display name of the field measurement |
| FieldMeasurement_Identifier | String | | Yes | No | Unique identifier code for the field measurement type |
| Section_Name | String | | Yes | No | Name of the section grouping this measurement |
| Section_Identifier | String | | Yes | No | Unique identifier for the section |
| Reading | String | | Yes | No | The captured measurement value as string (type interpretation via Data_Type) |
| Data_Type | Int64 | | No | No | Classification code: 0=Bool, 1=Date, 2=Time, 3=Text, 4=Number, 5=Location, 6=Signature, 7=Attachment, 8=MultiSelect, 9=Unknown, 10=Calculated |
| Lower_Boundary | Double | | Yes | No | Minimum acceptable value for numeric readings |
| Upper_Boundary | Double | | Yes | No | Maximum acceptable value for numeric readings |
| Captured_On | DateTime | yyyy-MM-dd HH:mm:ss | No | No | Timestamp when the measurement was captured |
| Completed_At | DateTime | yyyy-MM-dd HH:mm:ss | Yes | No | Timestamp when the measurement was marked complete |
| Comments | String | | Yes | No | User-entered comments about the measurement |
| Preface | String | | Yes | No | Text displayed before the measurement field |
| Postface | String | | Yes | No | Text displayed after the measurement field |
| Unit | String | | Yes | No | Unit of measurement (e.g., "kg", "°C", "psi") |
| Selected_Multi_Select_Value | String | | Yes | No | Selected option(s) for multi-select field measurements |
| Last_Updated | DateTime | yyyy-MM-dd HH:mm:ss | Yes | No | Timestamp of last modification |
| Created_Date | DateTime | yyyy-MM-dd HH:mm:ss | No | No | Creation timestamp (incremental refresh key) |
| Captured_On_Datekey | Int64 | YYYYMMDD | Yes | No | Date key derived from Captured_On timestamp |
| Completed_At_Datekey | Int64 | YYYYMMDD | Yes | No | Date key derived from Completed_At timestamp |
| Number_Of_Attachments | Int64 | | Yes | No | Count of attached files for this measurement |
| Number_Of_InlinePhotos | Int64 | | Yes | No | Count of inline photos captured with this measurement |
| LastLoaded | DateTime | yyyy-MM-dd HH:mm:ss | Yes | No | ETL timestamp for data lineage |

---

## Calculated Columns

### Captured_On_Hour
Rounds the `Captured_On` timestamp to the nearest 15-minute interval for time-based aggregation. Handles 24-hour edge case by converting "24:00" to "00:00".

```dax
Captured_On_Hour = 
VAR _CapturedOn_Hour = HOUR ( TimeSeries_FieldMeasurements[Captured_On] )
VAR _CapturedOn_Minute = MINUTE ( TimeSeries_FieldMeasurements[Captured_On] )
VAR _CapturedOn_Second = SECOND ( TimeSeries_FieldMeasurements[Captured_On] )
RETURN
    SWITCH (
        TRUE (),
        _CapturedOn_Second >= 1 && _CapturedOn_Minute >= 52, _CapturedOn_Hour + 1,
        _CapturedOn_Minute >= 38, _CapturedOn_Hour & ":45",
        _CapturedOn_Minute >= 23, _CapturedOn_Hour & ":30",
        _CapturedOn_Minute >= 8, _CapturedOn_Hour & ":15",
        _CapturedOn_Hour & ":00"
    )
```

### Completed_At_Hour
Rounds the `Completed_At` timestamp to the nearest 15-minute interval for completion time analysis. Uses same logic as `Captured_On_Hour`.

```dax
Completed_At_Hour = 
VAR _CompletedAt_Hour = HOUR ( TimeSeries_FieldMeasurements[Completed_At] )
VAR _CompletedAt_Minute = MINUTE ( TimeSeries_FieldMeasurements[Completed_At] )
VAR _CompletedAt_Second = SECOND ( TimeSeries_FieldMeasurements[Completed_At] )
RETURN
    SWITCH (
        TRUE (),
        _CompletedAt_Second >= 1 && _CompletedAt_Minute >= 52, _CompletedAt_Hour + 1,
        _CompletedAt_Minute >= 38, _CompletedAt_Hour & ":45",
        _CompletedAt_Minute >= 23, _CompletedAt_Hour & ":30",
        _CompletedAt_Minute >= 8, _CompletedAt_Hour & ":15",
        _CompletedAt_Hour & ":00"
    )
```

---

## Relationships

### Outbound Relationships
| To Table | From Column(s) | To Column(s) | Cardinality | Cross Filter | Relationship ID |
|----------|---------------|--------------|-------------|--------------|-----------------|
| `Dim_Tenant` | Tenant_Id | Tenant_Id | Many-to-One | Single | (Tenant context) |
| `TimeSeries` | TimeSeries_Id | Id | Many-to-One | Single | (Parent timeseries) |
| `Dim_Date_Reference` | Captured_On_Datekey | Date_Key | Many-to-One | Single | (Captured date) |
| `Dim_Date_Reference` | Completed_At_Datekey | Date_Key | Many-to-One | Single | (Completed date) |

### Inbound Relationships
| From Table | From Column(s) | To Column(s) | Cardinality | Cross Filter | Relationship ID |
|-----------|---------------|--------------|-------------|--------------|-----------------|
| `Assignment_FieldMeasurement_Exceptions` | FieldMeasurement_Id | Id | Many-to-One | Single | (Field measurement exceptions) |

---

## Power Query M Source

```m
let
    Source = Value.NativeQuery(
        Obzervr_DataWarehouse,
        "SELECT * FROM [dbo].[VW_TimeSeriesFieldMeasurements]"
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
            {"Id", Int64.Type},
            {"TimeSeries_Id", Int64.Type},
            {"FieldMeasurement_Name", type text},
            {"FieldMeasurement_Identifier", type text},
            {"Section_Name", type text},
            {"Section_Identifier", type text},
            {"Reading", type text},
            {"Data_Type", Int64.Type},
            {"Lower_Boundary", type number},
            {"Upper_Boundary", type number},
            {"Captured_On", type datetime},
            {"Completed_At", type datetime},
            {"Comments", type text},
            {"Preface", type text},
            {"Postface", type text},
            {"Unit", type text},
            {"Selected_Multi_Select_Value", type text},
            {"Last_Updated", type datetime},
            {"Created_Date", type datetime},
            {"Number_Of_Attachments", Int64.Type},
            {"Number_Of_InlinePhotos", Int64.Type},
            {"LastLoaded", type datetime}
        }
    ),
    #"Added Captured_On_Datekey" = Table.AddColumn(
        #"Changed Types",
        "Captured_On_Datekey",
        each Date.Year([Captured_On]) * 10000 
            + Date.Month([Captured_On]) * 100 
            + Date.Day([Captured_On]),
        Int64.Type
    ),
    #"Added Completed_At_Datekey" = Table.AddColumn(
        #"Added Captured_On_Datekey",
        "Completed_At_Datekey",
        each if [Completed_At] = null then null
            else Date.Year([Completed_At]) * 10000 
                + Date.Month([Completed_At]) * 100 
                + Date.Day([Completed_At]),
        Int64.Type
    ),
    #"Filtered by Incremental Refresh" = Table.SelectRows(
        #"Added Completed_At_Datekey",
        each [Created_Date] >= RangeStart and [Created_Date] < RangeEnd
    )
in
    #"Filtered by Incremental Refresh"
```

---

## DAX Query Patterns

### Example 1: Field Measurements by Data Type
```dax
EVALUATE
SUMMARIZECOLUMNS(
    TimeSeries_FieldMeasurements[Data_Type],
    "Measurement_Count", COUNTROWS(TimeSeries_FieldMeasurements),
    "Unique_Field_Types", DISTINCTCOUNT(TimeSeries_FieldMeasurements[FieldMeasurement_Identifier]),
    "Avg_Attachments", AVERAGE(TimeSeries_FieldMeasurements[Number_Of_Attachments])
)
ORDER BY TimeSeries_FieldMeasurements[Data_Type]
```

### Example 2: Out of Boundary Numeric Readings
```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        TimeSeries_FieldMeasurements,
        "Measurement_Name", TimeSeries_FieldMeasurements[FieldMeasurement_Name],
        "Reading", TimeSeries_FieldMeasurements[Reading],
        "Reading_Numeric", VALUE(TimeSeries_FieldMeasurements[Reading]),
        "Lower_Boundary", TimeSeries_FieldMeasurements[Lower_Boundary],
        "Upper_Boundary", TimeSeries_FieldMeasurements[Upper_Boundary],
        "Captured_On", TimeSeries_FieldMeasurements[Captured_On]
    ),
    TimeSeries_FieldMeasurements[Data_Type] = 4  // Numeric type
    && (
        VALUE(TimeSeries_FieldMeasurements[Reading]) < TimeSeries_FieldMeasurements[Lower_Boundary]
        || VALUE(TimeSeries_FieldMeasurements[Reading]) > TimeSeries_FieldMeasurements[Upper_Boundary]
    )
)
ORDER BY TimeSeries_FieldMeasurements[Captured_On] DESC
```

### Example 3: Measurement Completion Analysis
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_Date_Reference[Calendar_Year],
    Dim_Date_Reference[Calendar_Month_Name],
    "Total_Captured", COUNTROWS(
        FILTER(
            TimeSeries_FieldMeasurements,
            NOT ISBLANK(TimeSeries_FieldMeasurements[Captured_On])
        )
    ),
    "Total_Completed", COUNTROWS(
        FILTER(
            TimeSeries_FieldMeasurements,
            NOT ISBLANK(TimeSeries_FieldMeasurements[Completed_At])
        )
    ),
    "Completion_Rate", DIVIDE(
        COUNTROWS(FILTER(TimeSeries_FieldMeasurements, NOT ISBLANK([Completed_At]))),
        COUNTROWS(FILTER(TimeSeries_FieldMeasurements, NOT ISBLANK([Captured_On]))),
        0
    )
)
ORDER BY Dim_Date_Reference[Calendar_Year], Dim_Date_Reference[Month_Number]
```

### Example 4: Field Measurements with Comments and Photos
```dax
EVALUATE
TOPN(
    100,
    SELECTCOLUMNS(
        FILTER(
            TimeSeries_FieldMeasurements,
            NOT ISBLANK(TimeSeries_FieldMeasurements[Comments])
            || TimeSeries_FieldMeasurements[Number_Of_InlinePhotos] > 0
        ),
        "Assignment", RELATED(TimeSeries[Assignment_Id]),
        "Measurement_Name", TimeSeries_FieldMeasurements[FieldMeasurement_Name],
        "Reading", TimeSeries_FieldMeasurements[Reading],
        "Comments", TimeSeries_FieldMeasurements[Comments],
        "Photo_Count", TimeSeries_FieldMeasurements[Number_Of_InlinePhotos],
        "Attachment_Count", TimeSeries_FieldMeasurements[Number_Of_Attachments],
        "Captured_On", TimeSeries_FieldMeasurements[Captured_On]
    ),
    TimeSeries_FieldMeasurements[Captured_On],
    DESC
)
```

### Example 5: Hourly Capture Pattern Analysis
```dax
EVALUATE
SUMMARIZECOLUMNS(
    TimeSeries_FieldMeasurements[Captured_On_Hour],
    "Measurement_Count", COUNTROWS(TimeSeries_FieldMeasurements),
    "Unique_Sections", DISTINCTCOUNT(TimeSeries_FieldMeasurements[Section_Name]),
    "Avg_Time_To_Complete_Minutes", AVERAGE(
        DATEDIFF(
            TimeSeries_FieldMeasurements[Captured_On],
            TimeSeries_FieldMeasurements[Completed_At],
            MINUTE
        )
    )
)
ORDER BY TimeSeries_FieldMeasurements[Captured_On_Hour]
```

### Example 6: Multi-Select Field Distribution
```dax
EVALUATE
FILTER(
    SUMMARIZECOLUMNS(
        TimeSeries_FieldMeasurements[FieldMeasurement_Name],
        TimeSeries_FieldMeasurements[Selected_Multi_Select_Value],
        "Selection_Count", COUNTROWS(TimeSeries_FieldMeasurements)
    ),
    TimeSeries_FieldMeasurements[Data_Type] = 8  // Multi-select type
    && NOT ISBLANK(TimeSeries_FieldMeasurements[Selected_Multi_Select_Value])
)
ORDER BY [Selection_Count] DESC
```

---

## Data Model Pattern

### Field Measurement Fact Pattern
`TimeSeries_FieldMeasurements` implements a type-agnostic measurement storage pattern where the `Reading` column stores all measurement values as strings, and the `Data_Type` column classifies each reading for proper interpretation and conversion.

**Data Type Classification**:
- **0 = Boolean**: Reading values "true"/"false" or "1"/"0"
- **1 = Date**: Reading values in date format (convert with VALUE or DATEVALUE)
- **2 = Time**: Reading values in time format (convert with TIMEVALUE)
- **3 = Text**: Reading values are plain text strings
- **4 = Number**: Reading values are numeric (convert with VALUE)
- **5 = Location**: Reading values contain GPS coordinates or location identifiers
- **6 = Signature**: Reading values reference signature captures
- **7 = Attachment**: Reading values reference attached files
- **8 = Multi-Select**: Reading values contain comma-separated selected options (also stored in `Selected_Multi_Select_Value`)
- **9 = Unknown**: Reading type not classified
- **10 = Calculated**: Reading values computed from other measurements

**Boundary Enforcement**: For numeric readings (Data_Type = 4), the `Lower_Boundary` and `Upper_Boundary` columns define acceptable ranges. Values outside these boundaries can trigger exceptions (tracked in `Assignment_FieldMeasurement_Exceptions` table).

**Temporal Tracking**: The table captures two key timestamps:
- `Captured_On`: When the field technician initially recorded the measurement
- `Completed_At`: When the measurement was finalized or marked complete

The calculated hour columns (`Captured_On_Hour`, `Completed_At_Hour`) enable time-of-day analysis with 15-minute granularity, useful for identifying data capture patterns during shifts.

**Section Grouping**: Field measurements are organized into sections via `Section_Name` and `Section_Identifier`, allowing measurements to be grouped logically within the parent timeseries structure (e.g., "Pre-Start Checks", "Operational Readings", "Post-Operation").

**Relationship to TimeSeries**: Multiple field measurements link to a single timeseries instance (`TimeSeries_Id`). The timeseries acts as the container, while field measurements are the individual data points collected within that container.

**Example Scenario - Equipment Inspection**:
- TimeSeries instance: "Daily Haul Truck Pre-Start Inspection"
- Field Measurements within that timeseries:
  - "Engine Oil Level" (Data_Type=4, Reading="8.5", Unit="L", Lower_Boundary=7, Upper_Boundary=10)
  - "Tire Pressure Front Left" (Data_Type=4, Reading="95", Unit="psi", boundaries 90-100)
  - "Hydraulic Fluid Check" (Data_Type=0, Reading="true", represents pass/fail)
  - "Inspector Signature" (Data_Type=6, Reading=signature reference ID)
  - "Defect Photos" (Data_Type=7, Number_Of_InlinePhotos=3)
  - "Inspection Comments" (Data_Type=3, Reading="Minor oil leak observed", Comments="Scheduled for repair")

---

## Related Documentation
- **ERD_03_Field_Measurements.md** - ERD diagram showing relationship context
- **TimeSeries.md** - Parent container table documentation
- **Assignment_FieldMeasurement_Exceptions.md** - Exception tracking for out-of-range readings
- **Dim_Date_Reference.md** - Date dimension for temporal analysis
- **_Measures.md** - Centralized measures including reading-specific calculations

---

## Change History
| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-15 | Auto-generated | Initial documentation from TMDL metadata |

---

## Notes
- **Data Type Pattern**: The string-based `Reading` column with `Data_Type` classification enables flexible field measurement storage without requiring separate columns for each data type. DAX measures must convert readings based on the `Data_Type` value.
- **Boundary Validation**: Numeric readings (Data_Type=4) should be compared against `Lower_Boundary` and `Upper_Boundary` to identify out-of-specification measurements. Violations can be tracked in the `Assignment_FieldMeasurement_Exceptions` table.
- **Incremental Refresh**: 5-year rolling window based on `Created_Date` with 1-year increments provides balance between historical analysis and model size.
- **Hour Rounding Logic**: The 15-minute rounding in calculated hour columns uses a SWITCH statement that checks both minutes and seconds. If seconds >= 1 and minutes >= 52, the hour is incremented to handle "59:59" edge cases.
- **Date Key Calculation**: Both `Captured_On_Datekey` and `Completed_At_Datekey` are calculated in Power Query using the YYYYMMDD formula for efficient date relationships.
- **Multi-Select Values**: When `Data_Type` = 8, the `Selected_Multi_Select_Value` column contains the actual selected options, while `Reading` may contain a reference ID or summary.
- **Attachment Counts**: `Number_Of_Attachments` and `Number_Of_InlinePhotos` enable file tracking without requiring direct attachment table joins.
- **Preface and Postface**: These columns store template-defined text that appears before/after the measurement field in the Obzervr application UI.
- **Section Organization**: The `Section_Name` and `Section_Identifier` columns enable grouping of related measurements within a timeseries (e.g., grouping all safety checks, operational readings, or post-operation measurements).
- **Completion Tracking**: Not all measurements have a `Completed_At` timestamp. Some measurements may be captured immediately without a separate completion step.
