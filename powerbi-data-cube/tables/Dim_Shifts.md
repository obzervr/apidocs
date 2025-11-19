# Dim_Shifts

**Table Type:** Dimension Table  
**Schema:** dbo.DimShifts  
**Primary Key:** Id  
**Related ERD:** [ERD #2: Date Dimensions & Time Intelligence](../ERDs/ERD_02_Date_Dimensions_Time_Intelligence.md)

---

## Table Overview

Contains shift schedule information including shift names, start/end times, crew assignments, and shift codes. Supports operational planning and roster management by defining work periods with calendar dates and date keys for joining to date dimensions. Each shift record represents a scheduled work period with duration calculations.

**Source System:** Analytic Database (dbo.DimShifts)

**Row Count:** Varies by tenant and date range (typically 1,000-50,000 shifts)

**Refresh Type:** Full refresh with tenant filtering and soft delete removal

**Multi-Tenant:** Yes (filtered by Tenant_Id)

---

## Table Specifications

| Property | Value |
|----------|-------|
| Lineage Tag | 7e2865b8-6086-44b6-9f88-f6fcec5e6a1d |
| Query Group | Shift & Roster |
| Partitions | 1 (Dim_Shifts) |
| Relationships | 1 outbound to Dim_Tenant, relationships to date dimensions via Date_Key, inbound from Fact_Rosters |
| Calculated Columns | 1 (Shift_Hour) |
| Hierarchies | 0 |

---

## Column Specifications

| Column Name | Data Type | Format | Description | Source Column | Key | Nullable |
|-------------|-----------|--------|-------------|---------------|-----|----------|
| **Id** | string | - | Unique shift record identifier | Id | PK | No |
| **Calendar_Date** | datetime | General Date | Calendar date for the shift | CalendarDate | - | No |
| **Shift_Name** | string | - | Display name of shift (e.g., "Day Shift", "Night Shift") | ShiftName | - | Yes |
| **Shift_Start_Datetime** | datetime | General Date | Shift start date and time | ShiftStartDatetime | - | Yes |
| **Shift_End_Datetime** | datetime | General Date | Shift end date and time | ShiftEndDatetime | - | Yes |
| **Crew_Name** | string | - | Name of crew assigned to shift | CrewName | - | Yes |
| **Shift_Code** | string | - | Code identifier for shift type | ShiftCode | - | Yes |
| **Tenant_Id** | string | - | Tenant identifier | TenantId | FK | No |
| **LastUpdated** | datetime | General Date | Last update timestamp | LastUpdated | - | Yes |
| **CreatedDate** | datetime | General Date | Creation timestamp | CreatedDate | - | Yes |
| **Date_Key** | int64 | 0 | Integer date key (YYYYMMDD) from calendar date | - | FK | No |
| **Shift_Hour** | double | 0.00 | Shift duration in hours (calculated) | - | - | No |

---

## Calculated Columns

### Shift_Hour

**Purpose:** Calculate shift duration in hours from start and end datetimes

**Data Type:** Double (decimal hours)

**Format:** 0.00

**DAX Expression:**
```dax
Shift_Hour = 
VAR TotalMins = DATEDIFF ( Dim_Shifts[Shift_Start_Datetime], Dim_Shifts[Shift_End_Datetime], MINUTE ) 
RETURN
    IF (ISBLANK(Dim_Shifts[Shift_Start_Datetime]) || ISBLANK(Dim_Shifts[Shift_End_Datetime]), 12, DIVIDE(TotalMins,60) )
```

**Logic:**
1. Calculate total minutes between Shift_Start_Datetime and Shift_End_Datetime using DATEDIFF
2. If either datetime is blank, default to 12 hours
3. Otherwise, divide total minutes by 60 to get hours

**Use Cases:**
- Total shift hours reporting
- Shift duration analysis by shift type or crew
- Capacity planning (available work hours)
- Cost calculations based on shift duration

**Default Value:** 12 hours (standard shift duration) when start/end times are missing

---

## Relationships

### Outbound Relationships

**To Dim_Tenant**
- **Type:** Many-to-one
- **From Column:** Tenant_Id
- **To Column:** Dim_Tenant[Tenant_Id]
- **Cardinality:** Many:1
- **Cross-filter Direction:** Single
- **Status:** Active
- **Purpose:** Links shifts to tenant configuration

**To Dim_Date_Reference (via Date_Key)**
- **Type:** Many-to-one
- **From Column:** Date_Key
- **To Column:** Dim_Date_Reference[Date_Key]
- **Cardinality:** Many:1
- **Cross-filter Direction:** Single
- **Status:** Active
- **Purpose:** Links shifts to date dimension for calendar-based filtering

### Inbound Relationships

**From Fact_Rosters**
- **Type:** Many-to-one
- **From Column:** Fact_Rosters[Shift_Id]
- **To Column:** Dim_Shifts[Id]
- **Cardinality:** Many:1
- **Cross-filter Direction:** Single
- **Status:** Active
- **Purpose:** Links roster records to shift definitions

---

## Power Query M Source

```m
let
    Source = Sql.Database(#"SQL Server", #"Analytic Database", [CreateNavigationProperties=false, MultiSubnetFailover=true]),
    Table = Source{[Schema="dbo",Item="DimShifts"]}[Data],
    #"Filtered Rows" = Table.SelectRows(Table, each AllTenants = true or [TenantId] = #"TenantId1" or [TenantId] = TenantId2 or [TenantId] = TenantId3 or [TenantId] = TenantId4 or [TenantId] = TenantId5),
    #"Filtered Rows1" = Table.SelectRows(#"Filtered Rows", each [IsDeleted] = false),
    #"Removed Columns" = Table.RemoveColumns(#"Filtered Rows1",{"IsDeleted"}),
    #"Renamed Columns" = Table.RenameColumns(#"Removed Columns",{{"CalendarDate", "Calendar_Date"}, {"ShiftName", "Shift_Name"}, {"ShiftStartDatetime", "Shift_Start_Datetime"}, {"ShiftEndDatetime", "Shift_End_Datetime"}, {"CrewName", "Crew_Name"}, {"ShiftCode", "Shift_Code"}, {"TenantId", "Tenant_Id"}}),
    #"Added Custom" = Table.AddColumn(#"Renamed Columns", "Date_Key", each 10000 * Date.Year([Calendar_Date]) + 100 * Date.Month([Calendar_Date]) + Date.Day([Calendar_Date])),
    #"Changed Type" = Table.TransformColumnTypes(#"Added Custom",{{"Date_Key", Int64.Type}})
in
    #"Changed Type"
```

**Key Transformations:**
1. SQL database connection with MultiSubnetFailover
2. Tenant filtering using parameters
3. Soft delete filtering (IsDeleted = false)
4. Remove IsDeleted column from model
5. Column renaming to snake_case
6. Date_Key calculation (YYYYMMDD integer format)
7. Type conversion for Date_Key to Int64

---

## DAX Query Patterns

### Shift schedule by date range

```dax
EVALUATE
ADDCOLUMNS(
    FILTER(
        Dim_Shifts,
        [Calendar_Date] >= DATE(2025, 1, 1) && [Calendar_Date] <= DATE(2025, 1, 31)
    ),
    "Date", [Calendar_Date],
    "Shift", [Shift_Name],
    "Crew", [Crew_Name],
    "Start Time", [Shift_Start_Datetime],
    "End Time", [Shift_End_Datetime],
    "Duration (hrs)", [Shift_Hour]
)
ORDER BY [Calendar_Date], [Shift_Start_Datetime]
```

### Total shift hours by crew

```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_Shifts[Crew_Name],
    RELATED(Dim_Tenant[Tenant]),
    "Total Shifts", COUNTROWS(Dim_Shifts),
    "Total Hours", SUM(Dim_Shifts[Shift_Hour]),
    "Avg Hours per Shift", AVERAGE(Dim_Shifts[Shift_Hour])
)
ORDER BY [Total Hours] DESC
```

### Shift distribution by shift name

```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_Shifts[Shift_Name],
    Dim_Shifts[Shift_Code],
    "Count", COUNTROWS(Dim_Shifts),
    "Total Hours", SUM(Dim_Shifts[Shift_Hour]),
    "Min Duration", MIN(Dim_Shifts[Shift_Hour]),
    "Max Duration", MAX(Dim_Shifts[Shift_Hour]),
    "Avg Duration", AVERAGE(Dim_Shifts[Shift_Hour])
)
ORDER BY [Count] DESC
```

### Shifts with missing times

```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Dim_Shifts,
        "Shift ID", [Id],
        "Date", [Calendar_Date],
        "Shift", [Shift_Name],
        "Start", [Shift_Start_Datetime],
        "End", [Shift_End_Datetime],
        "Hours", [Shift_Hour]
    ),
    ISBLANK([Start]) || ISBLANK([End])
)
ORDER BY [Date]
```

### Monthly shift capacity

```dax
EVALUATE
ADDCOLUMNS(
    SUMMARIZECOLUMNS(
        RELATED(Dim_Date_Reference[Month_Year_Display]),
        RELATED(Dim_Date_Reference[Month_Year_Number]),
        Dim_Shifts[Crew_Name]
    ),
    "Total Shifts", COUNTROWS(Dim_Shifts),
    "Total Hours", SUM(Dim_Shifts[Shift_Hour]),
    "Avg Shifts per Day", 
        DIVIDE(
            COUNTROWS(Dim_Shifts),
            DISTINCTCOUNT(Dim_Shifts[Calendar_Date]),
            0
        )
)
ORDER BY [Month_Year_Number], [Crew_Name]
```

### Shift coverage by day of week

```dax
EVALUATE
ADDCOLUMNS(
    SUMMARIZECOLUMNS(
        RELATED(Dim_Date_Reference[Day_Name]),
        RELATED(Dim_Date_Reference[Day_of_Week])
    ),
    "Total Shifts", COUNTROWS(Dim_Shifts),
    "Total Hours", SUM(Dim_Shifts[Shift_Hour]),
    "Avg Hours per Day", AVERAGE(Dim_Shifts[Shift_Hour])
)
ORDER BY [Day_of_Week]
```

---

## Data Model Pattern

**Pattern:** Dimension Table with Calculated Duration

**Characteristics:**
- **Shift Definition**: Scheduled work periods with start/end times
- **Crew Assignment**: Links shifts to crew names for roster management
- **Date_Key Join**: Enables joining to date dimensions for time-based analysis
- **Duration Calculation**: Shift_Hour calculated from start/end datetimes
- **Soft Delete Filtering**: Only active shifts loaded (IsDeleted = false removed in Power Query)
- **Multi-Tenant**: Filtered by Tenant_Id

**Common Shift Patterns:**
- **Day Shift**: 06:00 - 18:00 (12 hours)
- **Night Shift**: 18:00 - 06:00 (12 hours)
- **Morning Shift**: 06:00 - 14:00 (8 hours)
- **Afternoon Shift**: 14:00 - 22:00 (8 hours)
- **Night Shift**: 22:00 - 06:00 (8 hours)

**Shift_Code Usage:**
Shift_Code provides standardized codes for shift types (e.g., "D" = Day, "N" = Night, "A" = Afternoon) enabling:
- Consistent shift classification across tenants
- Integration with external workforce management systems
- Simplified filtering and grouping in reports

**Calendar_Date vs Shift_Start_Datetime:**
- **Calendar_Date**: Business date for the shift (e.g., night shift starting Jan 17 at 22:00 has Calendar_Date = Jan 17)
- **Shift_Start_Datetime**: Actual start timestamp (e.g., Jan 17 22:00)
- **Date_Key**: Derived from Calendar_Date for joining to date dimensions

This distinction ensures shifts are attributed to the correct business day regardless of start time.

---

## Related Documentation

### ERD Documents
- [ERD #2: Date Dimensions & Time Intelligence](../ERDs/ERD_02_Date_Dimensions_Time_Intelligence.md) - Shift and date dimension relationships
- [ERD #7: Fact Tables & Audit](../ERDs/ERD_07_Fact_Tables_Audit.md) - Fact_Rosters table linking users to shifts

### Related Tables
- **Fact_Rosters** - Links users to shifts for roster assignments
- **Dim_Date_Reference** - Master date dimension (joined via Date_Key)
- **Dim_Tenant** - Tenant configuration

### Other Documentation
- [ERD_Overview.md](../ERD_Overview.md) - Date_Key pattern documentation
- [Dim_Date_Reference.md](Dim_Date_Reference.md) - Master date dimension

---

## Change History

| Date | Change Description | Modified By |
|------|-------------------|-------------|
| 2025-11-18 | Initial table documentation created from TMDL metadata | AI Documentation Generator |

---

## Notes

**Default Shift Duration:** 12 hours used as default when start/end times are blank, reflecting common 12-hour shift patterns in mining and industrial operations.

**Date_Key Generation:** Power Query calculates Date_Key from Calendar_Date using formula: Year * 10000 + Month * 100 + Day, producing YYYYMMDD integer format consistent with date dimension.

**Cross-Day Shifts:** Shifts spanning midnight (e.g., night shift 22:00-06:00) are attributed to the Calendar_Date they start on, ensuring consistent business day attribution.

**Crew_Name Usage:** Crew names enable roster reporting by crew, shift handover analysis, and crew capacity planning.

**Shift_Hour Precision:** Stored as double with 0.00 format, supporting fractional hours (e.g., 12.5 hours for 12h 30m shifts).

**Missing Times Handling:** When Shift_Start_Datetime or Shift_End_Datetime is blank, Shift_Hour defaults to 12 rather than BLANK(), ensuring duration calculations don't break measures. Consider data quality review if many shifts have missing times.

**Tenant-Specific Shifts:** Each tenant defines their own shift schedules. Shift_Name, Shift_Code, and Crew_Name may vary by tenant based on operational requirements.

**IsDeleted Filtering:** Soft-deleted shifts removed in Power Query (not retained in model), ensuring only active/scheduled shifts available for analysis.
