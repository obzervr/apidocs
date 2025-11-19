# Parameter_Asset_Hours_Type

## Table Overview
`Parameter_Asset_Hours_Type` is a field parameter table that enables dynamic measure selection for asset hours analysis. This table implements Power BI's field parameter pattern using the `NAMEOF()` function to reference measures, allowing users to switch between different hour/duration metrics through a slicer or visual interaction.

The parameter provides 6 options for analyzing asset hours: Actual Hours, Estimated Hours, Actual Duration, Estimated Duration, Actual Downtime, and Estimated Downtime.

**Current Status**: Calculated table using field parameter pattern with measure references.

---

## Specifications
- **Source**: DAX calculated table (inline row constructor)
- **Row Count**: 6 (one row per parameter option)
- **Grain**: One row per selectable measure option
- **Primary Key**: Not explicitly defined (display name serves as natural key)
- **Incremental Refresh**: Not applicable (calculated table)
- **Partitioning Strategy**: Single partition
- **Source Columns**: 3 (display name, measure reference, sort order)
- **Calculated Columns**: 0
- **Parameter Type**: Field Parameter (version 3, kind 2)

---

## Column Specifications

| Column Name | Data Type | Format | Nullable | Hidden | Description |
|------------|-----------|--------|----------|--------|-------------|
| Parameter_Asset_Hours_Type | String | | No | No | Display name for the parameter option (e.g., "Actual Hours") |
| Parameter_Asset_Hours_Type Fields | String | | No | No | NAMEOF reference to the corresponding measure (e.g., NAMEOF('_Measures'[Reading - Actual Hours])) |
| Parameter_Asset_Hours_Type Order | Numeric | 0 | No | Yes | Sort order integer (0-5) for consistent option sequence |

**Column Relationships**:
- `Parameter_Asset_Hours_Type` is sorted by `Parameter_Asset_Hours_Type Order`
- `Parameter_Asset_Hours_Type Fields` is sorted by `Parameter_Asset_Hours_Type Order`
- `Parameter_Asset_Hours_Type` is grouped by `Parameter_Asset_Hours_Type Fields` (field parameter linking)

---

## Calculated Columns
None. This table uses only inline-defined source columns.

---

## Relationships

### Outbound Relationships
None. Field parameter tables do not participate in model relationships.

### Inbound Relationships
None. Field parameter tables operate independently of the relationship model.

---

## DAX Table Source

```dax
Parameter_Asset_Hours_Type = 
{
    ("Actual Hours", NAMEOF('_Measures'[Reading - Actual Hours]), 0),
    ("Estimated Hours", NAMEOF('_Measures'[Reading - Estimated Hours]), 1),
    ("Actual Duration", NAMEOF('_Measures'[Reading - Actual Duration]), 2),
    ("Estimated Duration", NAMEOF('_Measures'[Reading - Estimated Duration]), 3),
    ("Actual Downtime", NAMEOF('_Measures'[Reading - Actual Downtime]), 4),
    ("Estimated Downtime", NAMEOF('_Measures'[Reading - Estimated Downtime]), 5)
}
```

**Table Structure**:
Each tuple defines one parameter option:
1. **Display Name** (column 1): User-friendly label shown in slicers/visuals
2. **Measure Reference** (column 2): `NAMEOF()` reference to measure in `_Measures` table
3. **Sort Order** (column 3): Integer for consistent ordering (0-5)

**NAMEOF Function**:
The `NAMEOF()` function returns the fully qualified name of the measure as a string. This enables dynamic measure selection without hard-coding measure names in visual-level logic.

---

## DAX Query Patterns

### Example 1: Parameter Options List
```dax
EVALUATE
SELECTCOLUMNS(
    Parameter_Asset_Hours_Type,
    "Option", Parameter_Asset_Hours_Type[Parameter_Asset_Hours_Type],
    "Measure_Reference", Parameter_Asset_Hours_Type[Parameter_Asset_Hours_Type Fields],
    "Sort_Order", Parameter_Asset_Hours_Type[Parameter_Asset_Hours_Type Order]
)
ORDER BY Parameter_Asset_Hours_Type[Parameter_Asset_Hours_Type Order]
```

### Example 2: Using Parameter in Measure
```dax
Selected_Asset_Hours = 
VAR SelectedMeasure = SELECTEDVALUE(Parameter_Asset_Hours_Type[Parameter_Asset_Hours_Type Fields])
RETURN
    SWITCH(
        SelectedMeasure,
        NAMEOF('_Measures'[Reading - Actual Hours]), [Reading - Actual Hours],
        NAMEOF('_Measures'[Reading - Estimated Hours]), [Reading - Estimated Hours],
        NAMEOF('_Measures'[Reading - Actual Duration]), [Reading - Actual Duration],
        NAMEOF('_Measures'[Reading - Estimated Duration]), [Reading - Estimated Duration],
        NAMEOF('_Measures'[Reading - Actual Downtime]), [Reading - Actual Downtime],
        NAMEOF('_Measures'[Reading - Estimated Downtime]), [Reading - Estimated Downtime],
        BLANK()
    )
```

### Example 3: Parameter with Asset Dimension
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_AssignmentPoints[Point_Name],
    "Selected_Metric", [Selected_Asset_Hours],
    "Metric_Type", SELECTEDVALUE(Parameter_Asset_Hours_Type[Parameter_Asset_Hours_Type])
)
ORDER BY [Selected_Metric] DESC
```

### Example 4: Comparing Parameter Options
```dax
EVALUATE
ADDCOLUMNS(
    Dim_AssignmentPoints,
    "Actual_Hours", [Reading - Actual Hours],
    "Estimated_Hours", [Reading - Estimated Hours],
    "Actual_Duration", [Reading - Actual Duration],
    "Estimated_Duration", [Reading - Estimated Duration],
    "Actual_Downtime", [Reading - Actual Downtime],
    "Estimated_Downtime", [Reading - Estimated Downtime],
    "Variance_Hours", [Reading - Actual Hours] - [Reading - Estimated Hours],
    "Variance_Duration", [Reading - Actual Duration] - [Reading - Estimated Duration]
)
```

---

## Data Model Pattern

### Field Parameter Pattern
`Parameter_Asset_Hours_Type` implements Power BI's field parameter pattern, enabling dynamic measure selection through user interaction:

**Pattern Characteristics**:
- **Inline Table Constructor**: Uses `{(...), (...)}` syntax to define parameter options
- **NAMEOF References**: Uses `NAMEOF()` to reference measures by string name
- **Three-Column Structure**: Display name + Measure reference + Sort order
- **Extended Property**: `ParameterMetadata` with `"kind": 2, "version": 3`
- **Related Column Details**: `groupByColumn` links display name to measure reference
- **Sort By Column**: Ensures consistent ordering via integer sort column

**Field Parameter Benefits**:
1. **Dynamic Measure Selection**: Users choose which measure to display without separate visuals
2. **Reduced Visual Count**: One visual serves multiple metrics
3. **Consistent Comparison**: Same visual formatting applies regardless of selected measure
4. **User Empowerment**: Self-service metric selection without modifying reports
5. **Cleaner Reports**: Fewer redundant visuals for similar metrics

**Parameter Usage Flow**:
1. **Slicer Creation**: Add `Parameter_Asset_Hours_Type` to a slicer visual
2. **Measure Reference**: Use `Parameter_Asset_Hours_Type Fields` column in chart/table
3. **User Selection**: User selects parameter option in slicer
4. **Dynamic Display**: Visual automatically shows selected measure's values

**Six Parameter Options**:

1. **Actual Hours** (Order 0): Actual asset operating hours from field measurements
2. **Estimated Hours** (Order 1): Estimated asset hours from planning/scheduling
3. **Actual Duration** (Order 2): Actual duration of work/activity
4. **Estimated Duration** (Order 3): Estimated duration from planning
5. **Actual Downtime** (Order 4): Actual equipment downtime hours
6. **Estimated Downtime** (Order 5): Estimated/planned downtime hours

**Actual vs Estimated Pattern**:
The parameter options follow an "actual vs estimated" comparison pattern:
- **Actual metrics**: Captured from field measurements and execution data
- **Estimated metrics**: Derived from planning, scheduling, or baseline data
- **Variance analysis**: Users can compare actual vs estimated to track accuracy

**Hours vs Duration vs Downtime**:
Three distinct time concepts:
- **Hours**: Total operating hours (asset runtime, meter readings)
- **Duration**: Work activity duration (assignment duration, shift duration)
- **Downtime**: Equipment non-operational time (maintenance, breakdowns)

**Measure Source**:
All referenced measures live in the `_Measures` table under `_Base Measure\Reading` display folder, derived from reading data in `TimeSeries_FieldMeasurements`.

**NAMEOF Advantages**:
Using `NAMEOF()` instead of hard-coded strings:
- **Refactoring Safety**: Renaming measures automatically updates parameter references
- **IntelliSense Support**: DAX editor provides measure name suggestions
- **Error Prevention**: Invalid measure names cause compilation errors, not runtime failures
- **Maintainability**: Clear connection between parameter and measures

**Extended Property Metadata**:
```json
{
  "kind": 2,
  "version": 3
}
```
- **kind: 2**: Indicates field parameter type (vs other parameter types)
- **version: 3**: Power BI field parameter version 3 (current implementation)

**Sort Order Strategy**:
The 0-5 integer sort ensures:
- Actual metrics appear before Estimated metrics
- Hours → Duration → Downtime progression
- Consistent user experience across all visuals

**Related Column Details**:
The `relatedColumnDetails` with `groupByColumn` configuration links the display name to measure reference, enabling Power BI to:
- Group by display name while showing measure values
- Maintain correct field parameter behavior in visuals
- Support drill-through and filtering operations

**Example Scenario - Asset Performance Dashboard**:

**Slicer Visual** - Parameter Selection:
- ☑ Actual Hours
- ☐ Estimated Hours
- ☐ Actual Duration
- ☐ Estimated Duration
- ☐ Actual Downtime
- ☐ Estimated Downtime

**Table Visual** - Asset Hours by Equipment:

| Equipment | Selected Metric (Actual Hours) |
|-----------|-------------------------------|
| Excavator EX-01 | 2,847 hrs |
| Haul Truck HT-05 | 1,923 hrs |
| Drill Rig DR-03 | 1,456 hrs |
| Loader LD-02 | 1,234 hrs |
| Grader GR-01 | 892 hrs |

**User changes slicer to "Estimated Hours"**:

| Equipment | Selected Metric (Estimated Hours) |
|-----------|----------------------------------|
| Excavator EX-01 | 2,920 hrs |
| Haul Truck HT-05 | 2,080 hrs |
| Drill Rig DR-03 | 1,560 hrs |
| Loader LD-02 | 1,300 hrs |
| Grader GR-01 | 960 hrs |

**Variance Analysis** (using both measures):

| Equipment | Actual Hours | Estimated Hours | Variance | Variance % |
|-----------|-------------|----------------|----------|-----------|
| Excavator EX-01 | 2,847 | 2,920 | -73 | -2.5% |
| Haul Truck HT-05 | 1,923 | 2,080 | -157 | -7.5% |
| Drill Rig DR-03 | 1,456 | 1,560 | -104 | -6.7% |
| Loader LD-02 | 1,234 | 1,300 | -66 | -5.1% |
| Grader GR-01 | 892 | 960 | -68 | -7.1% |

**Analysis**: All equipment showing lower actual hours than estimated, suggesting either:
- Conservative usage patterns
- Downtime not accounted in estimates
- Inaccurate estimation baseline

**User switches to "Actual Downtime"**:

| Equipment | Selected Metric (Actual Downtime) |
|-----------|----------------------------------|
| Haul Truck HT-05 | 157 hrs |
| Drill Rig DR-03 | 104 hrs |
| Excavator EX-01 | 73 hrs |
| Grader GR-01 | 68 hrs |
| Loader LD-02 | 66 hrs |

**Insight**: Haul Truck HT-05 has highest downtime, correlating with largest hours variance.

**Dashboard Benefit**:
- **Single Table Visual**: Serves 6 different metrics through parameter selection
- **User Flexibility**: Report consumers explore metrics without designer intervention
- **Consistent Format**: Same column width, formatting, conditional formatting regardless of selection
- **Report Simplicity**: 1 visual instead of 6 separate tables

---

## Related Documentation
- **ERD_08_Measures_Metadata.md** - ERD diagram showing measures and metadata relationship context
- **_Measures.md** - Centralized measures table containing referenced Reading measures
- **TimeSeries_FieldMeasurements.md** - Fact table source for reading data underlying these measures

---

## Change History
| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-15 | Auto-generated | Initial documentation from TMDL metadata |

---

## Notes
- **Field Parameter Pattern**: This table implements Power BI's field parameter pattern, enabling dynamic measure selection through slicer interaction.
- **NAMEOF Function**: Uses `NAMEOF()` to reference measures, providing refactoring safety and IntelliSense support compared to hard-coded strings.
- **Six Options**: Provides 6 asset hours metric options (Actual/Estimated × Hours/Duration/Downtime).
- **Actual vs Estimated**: Pattern supports variance analysis by offering parallel actual and estimated metrics.
- **Sort Order**: Integer sort column (0-5) ensures consistent parameter option ordering across visuals.
- **Three-Column Structure**: Display name + Measure reference + Sort order standard for field parameters.
- **Related Column Details**: `groupByColumn` configuration links display name to measure reference for proper field parameter behavior.
- **Extended Property**: `ParameterMetadata` with `kind: 2, version: 3` identifies this as field parameter version 3.
- **No Relationships**: Field parameter tables do not participate in model relationships - they operate independently.
- **Measure Source**: All referenced measures live in `_Measures` table under `_Base Measure\Reading` display folder.
- **Reading Data**: Measures derive from `TimeSeries_FieldMeasurements` fact table reading data.
- **Hours vs Duration vs Downtime**: Three distinct time concepts for different analysis purposes (operating hours, work duration, equipment downtime).
- **User Empowerment**: Enables self-service metric selection without report designer involvement.
- **Visual Simplification**: One visual with parameter serves multiple metrics, reducing report complexity.
- **Consistent Formatting**: Same visual formatting applies regardless of selected measure.
- **Variance Analysis Use Case**: Primary use case is comparing actual vs estimated to track planning accuracy.
- **Equipment Performance**: Commonly used for asset/equipment performance monitoring in mining/industrial contexts.
- **Inline Table Constructor**: Uses `{(...)}` syntax to define parameter options directly in DAX.
- **Sort By Column Configuration**: Both display column and fields column sorted by order column for consistency.
- **Version 3 Parameter**: Uses current Power BI field parameter version (version 3, kind 2).
- **No Incremental Refresh**: Calculated tables do not support incremental refresh.
- **Static Definition**: Parameter options are statically defined - adding new options requires DAX modification.
- **Alternative to Measure Switching**: More elegant solution than measure selection via disconnected tables or SWITCH measures.
