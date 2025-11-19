# _Measures

## Table Overview
`_Measures` is a calculated table that serves as a centralized container for 100+ DAX measures organized into display folders. This table follows the measures table pattern where a single-value placeholder table hosts all measures, providing a clean separation between data tables and business logic calculations.

The table contains measures across multiple functional areas: assignments, field measurements, readings, user activity, rosters, fragments, execution tracking, plant maintenance, usage analytics, and documentation helpers.

**Current Status**: Calculated table with single placeholder value. All measures organized in display folders for measure organization.

---

## Specifications
- **Source**: DAX calculated table
- **Row Count**: 1 (single BLANK() row placeholder)
- **Grain**: Single-row placeholder for measures
- **Primary Key**: Not applicable (measures table)
- **Incremental Refresh**: Not applicable (calculated table)
- **Partitioning Strategy**: Single partition
- **Source Columns**: 1 (Value - hidden placeholder)
- **Calculated Columns**: 0
- **Measures**: 100+ measures across 11 display folders

---

## Column Specifications

| Column Name | Data Type | Format | Nullable | Hidden | Description |
|------------|-----------|--------|----------|--------|-------------|
| Value | Numeric | 0 | Yes | Yes | Hidden placeholder column for measures table pattern |

---

## Calculated Columns
None. This table serves as a measures container only.

---

## Relationships

### Outbound Relationships
None. This is a measures-only table with no relationships.

### Inbound Relationships
None. The table exists solely to host measures.

---

## DAX Table Source

```dax
_Measures = {BLANK()}
```

The table creates a single-row placeholder using the `{BLANK()}` table constructor.

---

## Measures Organization

### Display Folder Structure
The 100+ measures are organized into the following display folders:

1. **_Misc** - Utility measures (card helpers, colors, timezone, refresh times, user info, RLS role)
2. **_Base Measure\Assignment** - Assignment core metrics (counts, overdue, rescheduled, duration, completion%)
3. **_Base Measure\Reading** - Reading measures by data type (Bool, Date, Time, Text, Number, Location, Signature, Attachment, MultiSelect)
4. **_Base Measure\Field Measurement** - Field measurement metrics (exceptions by priority)
5. **_Base Measure\User** - User activity metrics (logon counts, command counts)
6. **_Base Measure\Roster** - Roster metrics (planned hours, shift hours)
7. **_Base Measure\Fragment** - Fragment/template metrics (counts by type, captured/uncaptured fields)
8. **Execution** - Execution tracking (captured duration)
9. **Alias** - Label lookup measures (assignment point aliases)
10. **Plant Maintenance** - Plant maintenance specific metrics
11. **Usage** - Usage trend analysis
12. **Documentation** - Model documentation helpers (table/measure/relationship lists)

---

## Key Measures by Category

### _Misc - Utility Measures
- **Card - Current User Email**: `USERPRINCIPALNAME()`
- **Card - Current User RLS Role**: Returns RLS role name for current user
- **Card - Timezone Offset Hours**: `'Model Configuration'[Timezone_Offset_Hours]`
- **Card - Last Data Refresh (UTC)**: Model refresh timestamp
- **Card - Last Data Refresh (Local)**: Adjusted by timezone offset
- **Color - [various]**: Conditional formatting color codes for status/priority

### _Base Measure\Assignment
- **ASGT - Assignment Count**: `COUNTROWS(Assignments)`
- **ASGT - Assignment Overdue Count**: Assignments past due date
- **ASGT - Assignment Rescheduled Count**: Assignments with reschedule flag
- **ASGT - Assignment Duration Hours**: Total duration in hours
- **ASGT - Assignment Completion %**: Completion percentage calculation
- **ASGT - Active Assignment Count**: Assignments in active status
- **ASGT - Completed Assignment Count**: Assignments in completed status

### _Base Measure\Reading
Type-specific reading measures following data type pattern:
- **Reading - Bool Count**: Count of boolean readings (Data_Type = 0)
- **Reading - Date Count**: Count of date readings (Data_Type = 1)
- **Reading - Time Count**: Count of time readings (Data_Type = 2)
- **Reading - Text Count**: Count of text readings (Data_Type = 3)
- **Reading - Number Count**: Count of number readings (Data_Type = 4)
- **Reading - Number Sum**: Sum of numeric reading values
- **Reading - Location Count**: Count of location readings (Data_Type = 5)
- **Reading - Signature Count**: Count of signature readings (Data_Type = 6)
- **Reading - Attachment Count**: Count of attachment readings (Data_Type = 7)
- **Reading - MultiSelect Count**: Count of multi-select readings (Data_Type = 10)
- **Reading - Actual Hours**: Asset actual hours from readings
- **Reading - Estimated Hours**: Asset estimated hours from readings
- **Reading - Actual Duration**: Calculated actual duration
- **Reading - Estimated Duration**: Calculated estimated duration
- **Reading - Actual Downtime**: Actual downtime from readings
- **Reading - Estimated Downtime**: Estimated downtime from readings

### _Base Measure\Field Measurement
- **FM - Field Measurement Count**: `COUNTROWS(TimeSeries_FieldMeasurements)`
- **FM - TimeSeries Count**: `COUNTROWS(TimeSeries)`
- **FM - Exception Count**: Total exception count
- **FM - Exception High Priority Count**: High priority exceptions
- **FM - Exception Medium Priority Count**: Medium priority exceptions
- **FM - Exception Low Priority Count**: Low priority exceptions
- **FM - Exception Resolved Count**: Resolved exception count
- **FM - Exception Unresolved Count**: Unresolved exception count

### _Base Measure\User
- **User - LogOn Count**: `COUNTROWS(Fact_User_LogOn_Activities)`
- **User - Command Count**: `COUNTROWS(Fact_User_Assignment_Audit)`
- **User - Active User Count**: Distinct users with activity
- **User - Unique User Count**: `DISTINCTCOUNT(Dim_User_Reference[User_Id])`

### _Base Measure\Roster
- **Roster - Planned Shift Hour**: Sum of `Fact_Rosters[Planned_Shift_Hour]`
- **Roster - Shift Hour**: Sum of `Fact_Rosters[Shift_Hour]`
- **Roster - Planned Attendance**: Sum of planned attendance doubles
- **Roster - Planned Absence**: Sum of planned absence doubles
- **Roster - Roster Count**: `COUNTROWS(Fact_Rosters)`

### _Base Measure\Fragment
- **Fragment - Count by Fragment Type**: SWITCH statement handling WorkTemplate/GroupFragment/SectionFragment/FieldFragment counts with USERELATIONSHIP
- **Fragment - Fragment Details Row Count**: Total fragment detail records
- **Fragment - Fragment Details Group Count**: Distinct group count
- **Fragment - Fragment Details Section Count**: Distinct section count
- **Fragment - Is Field Captured**: String matching logic comparing captured field measurements to fragment fields (complex CONTAINSSTRING pattern)
- **Fragment - Uncaptured Fragment Details Row Count**: Count of uncaptured fields
- **Fragment - Count by Published WorkTemplate**: Count filtered to published work templates
- **Fragment - Count by Published Group Fragment**: Count filtered to published group fragments
- **Fragment - Is Assignment using Published WorkTemplate**: Boolean check via `Dim_WorkTemplates[Is_Published]`
- **Fragment - Is Assignment using Published Group Fragment**: Boolean check via `Dim_Fragments[Is_Published]`

### Execution
- **Execution - Captured Duration Hours**: Likely sum of execution duration

### Alias
- **Alias - Assignment Point**: Label lookup using assignment point hierarchy

### Plant Maintenance
Plant maintenance specific measures (specific measures not detailed in TMDL excerpt)

### Usage
Usage trend measures (specific measures not detailed in TMDL excerpt)

### Documentation
- **Doc - Related Measures by Selected_Tables**: CONCATENATEX filtering `Semantic_Model_Measures` where measure expression contains selected table name (uses SEARCH function)
- **Doc - Related Relationships by Selected_Tables**: CONCATENATEX filtering `Semantic_Model_Relationships` where FromTable or ToTable matches selected table

---

## Measure Pattern Examples

### Example 1: Type-Specific Reading Count Pattern
```dax
Reading - Bool Count = 
CALCULATE(
    COUNTROWS(TimeSeries_FieldMeasurements),
    TimeSeries_FieldMeasurements[Data_Type] = 0
)
```

This pattern repeats for each data type (0-10), enabling type-specific analysis of field measurements.

### Example 2: Fragment Field Capture Detection
```dax
Fragment - Is Field Captured = 
// Complex string matching logic
VAR FMNameList =
    CONCATENATEX(
        VALUES(TimeSeries_FieldMeasurements[FieldMeasurement_Name]),
        TimeSeries_FieldMeasurements[FieldMeasurement_Name] & " ",
        ", "
    )
VAR FMCheck = CONTAINSSTRING(FMNameList, MAX(Fact_Fragment_Details[Field_Name]) & " ")

VAR SecNameList =
    CONCATENATEX(
        VALUES(TimeSeries_FieldMeasurements[Section_Name]),
        TimeSeries_FieldMeasurements[Section_Name] & " ",
        ", "
    )
VAR SecCheck = CONTAINSSTRING(SecNameList, MAX(Fact_Fragment_Details[Section_Name]) & " ")

VAR GrpNameList =
    CONCATENATEX(
        VALUES(TimeSeries[Series_Name]),
        TimeSeries[Series_Name] & " ",
        ", "
    )
VAR GrpCheck = CONTAINSSTRING(GrpNameList, MAX(Fact_Fragment_Details[Group_Name]) & " ")

RETURN
    IF(
        FMCheck && SecCheck && GrpCheck,
        1,
        0
    )
```

This measure performs three-level string matching (Group→Section→Field) to detect if a fragment field has been captured in actual data.

### Example 3: Documentation Helper Pattern
```dax
Doc - Related Measures by Selected_Tables = 
CONCATENATEX(
    FILTER(
        Semantic_Model_Measures,
        SUMX(
            VALUES(Semantic_Model_Tables[Name]),
            IF(
                SEARCH(
                    Semantic_Model_Tables[Name],
                    Semantic_Model_Measures[Expression],
                    ,
                    0
                ) > 0,
                1,
                0
            )
        ) > 0
    ),
    Semantic_Model_Measures[Name],
    UNICHAR(10)
)
```

This measure searches measure expressions for table references, enabling dynamic documentation of measure dependencies.

### Example 4: Fragment Count with Dynamic Relationships
```dax
Fragment - Count by Fragment Type = 
SWITCH(
    SELECTEDVALUE(Dim_Fragment_Type[Fragment_Type]),
    "WorkTemplate",
        CALCULATE(
            COUNT(Dim_Published_Worktemplates[Template_Link]),
            FILTER(
                Fact_Fragment_Details,
                NOT(ISBLANK(Fact_Fragment_Details[WorkTemplate_Id]))
            ),
            USERELATIONSHIP(Dim_Fragment_Type[Fragment_Type], Dim_Published_Worktemplates[Fragment_Type])
        ),
    "GroupFragment",
        CALCULATE(
            COUNT(Dim_Published_Group_Fragments[Template_Link]),
            FILTER(
                Fact_Fragment_Details,
                NOT(ISBLANK(Fact_Fragment_Details[Group_Id]))
            ),
            USERELATIONSHIP(Dim_Fragment_Type[Fragment_Type], Dim_Published_Group_Fragments[Fragment_Type])
        ),
    "SectionFragment",
        CALCULATE(
            COUNT(Dim_Published_Section_Fragments[Template_Link]),
            FILTER(
                Fact_Fragment_Details,
                NOT(ISBLANK(Fact_Fragment_Details[Section_Id]))
            ),
            USERELATIONSHIP(Dim_Fragment_Type[Fragment_Type], Dim_Published_Section_Fragments[Fragment_Type])
        ),
    "FieldFragment",
        CALCULATE(
            COUNT(Dim_Published_Field_Fragments[Template_Link]),
            FILTER(
                Fact_Fragment_Details,
                NOT(ISBLANK(Fact_Fragment_Details[Field_Id]))
            ),
            USERELATIONSHIP(Dim_Fragment_Type[Fragment_Type], Dim_Published_Field_Fragments[Fragment_Type])
        )
)
```

This measure uses USERELATIONSHIP to activate different inactive relationships based on fragment type selection.

---

## Data Model Pattern

### Centralized Measures Table Pattern
`_Measures` implements the centralized measures table pattern, a common Power BI best practice for organizing business logic:

**Pattern Characteristics**:
- **Single Placeholder Row**: Table contains only `{BLANK()}` to satisfy table requirements
- **Hidden Value Column**: The `Value` column is hidden and never used
- **Measure Container**: All measures live in this table rather than scattered across fact/dimension tables
- **Display Folder Organization**: Measures organized by functional area for discoverability
- **No Relationships**: The table has no relationships - measures reference other tables explicitly

**Benefits of This Pattern**:
1. **Logical Organization**: Measures grouped by business function, not table structure
2. **Easier Discovery**: Users find related measures in one location
3. **Cleaner Model**: Separates calculations from data storage
4. **Reusability**: Measures can reference any table without relationship constraints
5. **Maintenance**: Centralized location simplifies measure management

**Display Folder Hierarchy**:
The measures use a two-level hierarchy:
- **Top Level**: `_Misc`, `_Base Measure`, `Execution`, `Alias`, `Plant Maintenance`, `Usage`, `Documentation`
- **Second Level**: Under `_Base Measure`: `\Assignment`, `\Reading`, `\Field Measurement`, `\User`, `\Roster`, `\Fragment`

The underscore prefix (`_`) pushes these folders to the top of alphabetical lists in the field well.

**Naming Conventions**:
Measures follow consistent prefixes:
- **ASGT -**: Assignment measures
- **FM -**: Field Measurement measures
- **Reading -**: Reading/data type measures
- **User -**: User activity measures
- **Roster -**: Roster/scheduling measures
- **Fragment -**: Fragment/template measures
- **Card -**: Card visual utility measures
- **Color -**: Conditional formatting measures
- **Doc -**: Documentation helper measures
- **Alias -**: Label lookup measures
- **Execution -**: Execution tracking measures

**Measure Complexity Spectrum**:
- **Simple Aggregations**: `COUNTROWS()`, `SUM()`, `DISTINCTCOUNT()` (most _Base Measure examples)
- **Filtered Calculations**: Using `CALCULATE()` with filters (overdue, by status, by data type)
- **Complex Logic**: SWITCH statements, USERELATIONSHIP, string matching (Fragment measures)
- **Documentation Helpers**: DMV-based measures querying model metadata (Doc measures)

**Type-Specific Reading Measures**:
The reading measures follow the type-agnostic storage pattern from `TimeSeries_FieldMeasurements`:
- Data_Type 0 = Bool → `Reading - Bool Count`
- Data_Type 1 = Date → `Reading - Date Count`
- Data_Type 2 = Time → `Reading - Time Count`
- Data_Type 3 = Text → `Reading - Text Count`
- Data_Type 4 = Number → `Reading - Number Count`, `Reading - Number Sum`
- Data_Type 5 = Location → `Reading - Location Count`
- Data_Type 6 = Signature → `Reading - Signature Count`
- Data_Type 7 = Attachment → `Reading - Attachment Count`
- Data_Type 10 = MultiSelect → `Reading - MultiSelect Count`

This enables type-specific analysis while maintaining single fact table storage.

**Fragment Capture Detection**:
The `Fragment - Is Field Captured` measure implements sophisticated string matching to detect if fragment fields defined in templates have been captured in actual field measurement data. This involves:
1. Building concatenated lists of field/section/group names from captured data
2. Checking if fragment detail names exist in those lists
3. Requiring all three levels (Group + Section + Field) to match
4. Returning 1/0 boolean for visualization

**Documentation Helpers**:
The `Doc -` measures enable dynamic model documentation by querying DMV tables:
- Search measure expressions for table references
- Find relationships involving selected tables
- Enable "what measures use this table?" and "what tables relate to this?" queries

**User Context Measures**:
Several measures provide user-specific context:
- **Card - Current User Email**: `USERPRINCIPALNAME()` for email display
- **Card - Current User RLS Role**: Shows which RLS role applies to current user
- Enables personalized dashboards and debugging RLS configuration

**Timezone Handling**:
Timezone measures enable local time display:
- **Card - Timezone Offset Hours**: Configured offset (e.g., +10 for AEST)
- **Card - Last Data Refresh (Local)**: UTC refresh time adjusted to local timezone
- Pattern: `[UTC Timestamp] + [Timezone_Offset_Hours]/24`

**Example Scenario - Measure Usage in Assignment Dashboard**:

**Assignment Overview Card**:
- Total Assignments: `[ASGT - Assignment Count]` = 1,247
- Active: `[ASGT - Active Assignment Count]` = 823
- Overdue: `[ASGT - Assignment Overdue Count]` = 156
- Completion Rate: `[ASGT - Assignment Completion %]` = 68.2%

**Field Measurement Card**:
- Total Measurements: `[FM - Field Measurement Count]` = 45,892
- Number Readings: `[Reading - Number Count]` = 18,234
- Text Readings: `[Reading - Text Count]` = 12,456
- Attachments: `[Reading - Attachment Count]` = 2,341

**User Activity Card**:
- Active Users: `[User - Active User Count]` = 247
- Total Logons: `[User - LogOn Count]` = 8,923
- Commands Executed: `[User - Command Count]` = 34,567

**Fragment Template Card**:
- Uncaptured Fields: `[Fragment - Uncaptured Fragment Details Row Count]` = 23
- Total Fragment Fields: `[Fragment - Fragment Details Row Count]` = 456

**System Info Card**:
- Current User: `[Card - Current User Email]` = john.smith@company.com
- User Role: `[Card - Current User RLS Role]` = Team User
- Last Refresh: `[Card - Last Data Refresh (Local)]` = 2024-11-08 06:15 AM
- Timezone: `[Card - Timezone Offset Hours]` = +10 (AEST)

---

## Related Documentation
- **ERD_08_Measures_Metadata.md** - ERD diagram showing measures and metadata relationship context
- **Parameter_Asset_Hours_Type.md** - Field parameter table used with Reading measures
- **Semantic_Model_Relationships.md** - DMV table queried by Doc measures
- **TimeSeries_FieldMeasurements.md** - Fact table analyzed by Reading measures
- **Assignments.md** - Fact table analyzed by ASGT measures
- **Fact_User_LogOn_Activities.md** - Fact table analyzed by User measures
- **Fact_Rosters.md** - Fact table analyzed by Roster measures
- **Fact_Fragment_Details.md** - Fact table analyzed by Fragment measures

---

## Change History
| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-15 | Auto-generated | Initial documentation from TMDL metadata |

---

## Notes
- **Centralized Measures Table**: This table implements the measures table pattern with 100+ measures organized in display folders, providing logical separation of business logic from data tables.
- **Single Placeholder Row**: The table contains only `{BLANK()}` as a placeholder - the `Value` column is hidden and never used in calculations.
- **No Relationships**: The measures table has no relationships to other tables. Measures reference other tables explicitly in DAX expressions.
- **Display Folder Organization**: Measures organized into 11 display folders (_Misc, _Base Measure with 6 subfolders, Execution, Alias, Plant Maintenance, Usage, Documentation) for logical grouping.
- **Naming Convention Prefixes**: Consistent prefixes enable measure discovery (ASGT-, FM-, Reading-, User-, Roster-, Fragment-, Card-, Color-, Doc-, Alias-, Execution-).
- **Type-Specific Reading Measures**: Separate measures for each Data_Type value (0-10) enable type-specific analysis of field measurements while maintaining single fact table.
- **Fragment Capture Detection**: Complex `Fragment - Is Field Captured` measure uses three-level string matching (Group+Section+Field) to detect if template fields have been captured in actual data.
- **Documentation Helpers**: `Doc -` measures query DMV tables to enable dynamic model documentation (which measures use a table, which tables relate to selected table).
- **User Context Measures**: `USERPRINCIPALNAME()` and RLS role detection enable personalized dashboards and RLS debugging.
- **Timezone Support**: Measures provide both UTC and local time display using configurable timezone offset from Model Configuration table.
- **USERELATIONSHIP Pattern**: Fragment measures use `USERELATIONSHIP()` to activate different inactive relationships based on fragment type selection.
- **Measure Complexity Levels**: Ranges from simple `COUNTROWS()` to complex SWITCH with USERELATIONSHIP to sophisticated string matching with CONCATENATEX + CONTAINSSTRING.
- **Base Measure Convention**: The `_Base Measure` folder contains foundational metrics, while top-level folders contain derived or specialized measures.
- **Color Measures**: Multiple `Color -` measures return hex codes for conditional formatting by status, priority, or other dimensions.
- **Card Measures**: `Card -` measures provide single-value outputs optimized for card visuals (current user, refresh times, configuration values).
- **Reading Asset Hours**: Six measures track actual vs estimated hours/duration/downtime from reading data, used with Asset Hours field parameter.
- **Exception Priority Measures**: Field measurement exceptions tracked by priority level (High/Medium/Low) and resolution status (Resolved/Unresolved).
- **Fragment Published vs Draft**: Measures distinguish between published and draft versions of work templates and group fragments.
- **Uncaptured Field Tracking**: Enables identification of template fields defined but not yet captured in actual field measurement data.
- **Active vs Total Counts**: Many measures have both total count and active/filtered count variants (assignments, users, exceptions).
- **SUMX with SEARCH Pattern**: Documentation helpers use `SUMX` with `SEARCH()` to detect table name occurrences in measure expressions.
- **UNICHAR(10) Delimiter**: Documentation measures use line feed character for multi-line text output in visuals.
- **Model Refresh Timestamp**: Last data refresh measures enable data currency monitoring and ETL validation.
- **Underscore Prefix Sorting**: Leading underscore (`_`) in folder names pushes them to top of alphabetical field lists for prominent placement.
