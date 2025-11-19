# Relative_Dates

**Table Type:** Bridge Table (Many-to-Many)  
**Schema:** Power Query M  
**Primary Key:** Composite (Period_Id + Date_Key)  
**Related ERD:** [ERD #2: Date Dimensions & Time Intelligence](../ERDs/ERD_02_Date_Dimensions_Time_Intelligence.md)

---

## Table Overview

Bridge table enabling relative date filtering with pre-defined date ranges (Today, Yesterday, Last 7 Days, This Month, Current Fiscal Year, etc.). Contains one row per date within each relative period, linking Period_Id to Date_Key. Implements many-to-many relationships with bidirectional cross-filtering to enable dynamic relative date slicers.

**Source System:** Power Query M with dynamic date calculations

**Row Count:** Variable (depends on date range and number of periods; typically 10,000-20,000 rows)

**Refresh Type:** Full refresh (recalculates relative dates based on current date)

**Multi-Tenant:** Yes (TodaysDate adjusted by tenant timezone offset)

**Bidirectional Filtering:** Yes (enables Period → Date and Date → Period filtering)

---

## Table Specifications

| Property | Value |
|----------|-------|
| Lineage Tag | bdc42131-dba9-45e8-8ed6-6f4d5aca14ac |
| Query Group | Data & Time |
| Partitions | 1 (Relative_Dates - Power Query M) |
| Relationships | 2 outbound (to Dim_Date_Reference and Dim_Relative_Dates_Type) |
| Calculated Columns | 0 |
| Hierarchies | 0 |

---

## Column Specifications

| Column Name | Data Type | Description | Sorted By | Nullable |
|-------------|-----------|-------------|-----------|----------|
| **Period** | string | Relative period name (e.g., "Today", "Last 30 Days") | Period_Id | No |
| **Period_Id** | double | Numeric identifier for the relative period | - | No |
| **Date_Key** | int64 | Integer date key (YYYYMMDD) linking to date dimension | - | No |

**Note:** Period column sorted by Period_Id ensures logical ordering in slicers (Today → Yesterday → This Week → etc.)

---

## Calculated Columns

This table has no calculated columns. All columns are generated via Power Query M transformations.

---

## Relationships

### Outbound Relationships

**To Dim_Date_Reference**
- **Type:** Many-to-one
- **From Column:** Date_Key
- **To Column:** Dim_Date_Reference[Date_Key]
- **Cardinality:** Many:1
- **Cross-filter Direction:** Both (bidirectional)
- **Status:** Active
- **Purpose:** Links relative dates to master date dimension for filtering

**To Dim_Relative_Dates_Type**
- **Type:** Many-to-one
- **From Column:** Period
- **To Column:** Dim_Relative_Dates_Type[Period]
- **Cardinality:** Many:1
- **Cross-filter Direction:** Single
- **Status:** Active
- **Purpose:** Links to period definitions and metadata

---

## Power Query M Source

```m
let
  // Change the Offset Parameter as per Tenant Timezone Offset
  OffsetMinutes = if Table.IsEmpty(Dim_Tenant) then 0 else Dim_Tenant{0}[Offset Minutes],
  OffsetHours = OffsetMinutes/60,
  TodaysDate =  Date.From(DateTimeZone.FixedUtcNow() + #duration(0,0,OffsetMinutes,0)),
  //Date.From(DateTimeZone.SwitchZone(DateTimeZone.FixedUtcNow(), OffsetHours)),
    Ranges = {
    {
        "All", 
        Date.From(Text.From(StartYear) & "01" & "01" ), 
        Date.From(Text.From(EndYear) & "12" & "31") , 
        1
    },      
    {
        "Today", 
        TodaysDate, 
        TodaysDate, 
        1
    }, 
    {
        "Yesterday", 
        Date.AddDays(TodaysDate, - 1), 
        Date.AddDays(TodaysDate, - 1), 
        2
    }, 
    {
        "This Week", 
        Date.StartOfWeek(TodaysDate, 1), 
        Date.EndOfWeek(TodaysDate, Day.Monday), 
        3
    }, 
    {
        "Current Week to Date", 
        Date.StartOfWeek(TodaysDate, 1), 
        TodaysDate, 
        4
    },
    {
        "Previous Week", 
        Date.AddWeeks(Date.StartOfWeek(TodaysDate, Day.Monday), - 1), 
        Date.StartOfWeek(TodaysDate, Day.Sunday), 
         5
    }, 
    {
        "Last 14 Days", 
        Date.AddDays(TodaysDate, - 13), 
        TodaysDate, 
         6
    }, 
    {
        "Last 30 Days", 
        Date.AddDays(TodaysDate, - 29), 
        TodaysDate, 
         6
    }, 
    {
        "Last 60 Days", 
        Date.AddDays(TodaysDate, - 59), 
        TodaysDate, 
         6
    },         
    {
        "This Month", 
        Date.StartOfMonth(TodaysDate), 
        Date.EndOfMonth(TodaysDate), 
        7
    }, 
    {
        "Current Month to Date", 
        Date.StartOfMonth(TodaysDate), 
        TodaysDate, 
        8
    },
    {
         "Previous Month", 
         Date.AddMonths(Date.StartOfMonth(TodaysDate), - 1), 
         Date.AddDays(Date.StartOfMonth(TodaysDate), - 1), 
         9
    }, 
    {
         "Last 3 Month", 
         Date.AddMonths(Date.StartOfMonth(TodaysDate), - 3), 
         TodaysDate, 
         9
    },
    {
         "Last 6 Month", 
         Date.AddMonths(Date.StartOfMonth(TodaysDate), - 6), 
         TodaysDate, 
         9
    }, 
    {
         "Last 12 Month", 
         Date.AddMonths(Date.StartOfMonth(TodaysDate), - 12), 
         TodaysDate, 
         9
    },         
    {
        "This Year", 
        Date.StartOfYear(TodaysDate), 
        Date.EndOfYear(TodaysDate),
        10
    },  
    {
        "This Fiscal Year", 
        if Date.Month(TodaysDate) > 6  then Date.From(Text.From(Date.Year(TodaysDate)) & "07" & "01")  else if Date.Month(TodaysDate) < 7 then Date.From(Text.From(Date.Year(TodaysDate)-1) & "07" & "01") else Date.From(Text.From(Date.Year(TodaysDate)-1) & "07" & "01"), 
        if Date.Month(TodaysDate) > 6  then Date.From(Text.From(Date.Year(TodaysDate)+1) & "06" & "30")  else if Date.Month(TodaysDate) < 7 then Date.From(Text.From(Date.Year(TodaysDate)) & "06" & "30") else Date.From(Text.From(Date.Year(TodaysDate)) & "06" & "30"),
        10
    },     
    {
        "Current Year To Date", 
        Date.StartOfYear(TodaysDate), 
        TodaysDate, 
        11
    }, 
    {
        "Current Fiscal Year To Date", 
        if Date.Month(TodaysDate) > 6  then Date.From(DateTimeZone.From(#date(Date.Year(TodaysDate),07,01)))  else if Date.Month(TodaysDate) < 7 then Date.From(DateTimeZone.From(#date(Date.Year(TodaysDate) - 1,07,01))) else Date.From(DateTimeZone.From(#date(Date.Year(TodaysDate) - 1,07,01))), 
        TodaysDate,
        11
    },    
    {
        "Previous Year", 
        Date.AddYears(Date.StartOfYear(TodaysDate), - 1), 
        Date.AddYears(Date.EndOfYear(TodaysDate), - 1), 
        12
    },
    {
        "Previous Fiscal Year", 
        if Date.Month(TodaysDate) > 6  then Date.AddYears(Date.From(Text.From(Date.Year(TodaysDate)) & "07" & "01"),-1)  else if Date.Month(TodaysDate) < 7 then Date.From(Text.From(Date.Year(TodaysDate)-2) & "07" & "01") else Date.From(Text.From(Date.Year(TodaysDate)-2) & "07" & "01"), 
        if Date.Month(TodaysDate) > 6  then Date.AddYears(Date.From(Text.From(Date.Year(TodaysDate)) & "06" & "30"),0)  else if Date.Month(TodaysDate) < 7 then Date.From(Text.From(Date.Year(TodaysDate)-1) & "06" & "30") else Date.From(Text.From(Date.Year(TodaysDate)-1) & "06" & "30"),
        12
    }
  },

  
  GetTables = List.Transform(Ranges, each Fn_Create_Dates(_{0}, _{1}, _{2}, _{3})),
  Output = Table.Combine(GetTables),
    #"Changed Type" = Table.TransformColumnTypes(Output,{{"Date", type date}}),
  Datekey = Table.AddColumn(#"Changed Type", "DateKey", each Date.Year ( [Date] ) * 10000
            + Date.Month ( [Date] ) * 100
            + Date.Day ( [Date] ), type number),
    #"Renamed Columns" = Table.RenameColumns(Datekey,{{"Sort", "Period_Id"}, {"DateKey", "Date_Key"}}),
    #"Removed Columns" = Table.RemoveColumns(#"Renamed Columns",{"Date"}),
    #"Changed Type1" = Table.TransformColumnTypes(#"Removed Columns",{{"Date_Key", Int64.Type}})
in
    #"Changed Type1"
```

**Key Transformations:**
1. **Tenant Timezone Adjustment**: TodaysDate adjusted by tenant's offset minutes from Dim_Tenant
2. **Range Definitions**: 22 pre-defined relative date ranges with start date, end date, and Period_Id
3. **Date Expansion**: Fn_Create_Dates function expands each range into individual date rows
4. **Date_Key Generation**: Converts dates to YYYYMMDD integer format
5. **Column Renaming**: Standardizes column names to snake_case

**Relative Date Periods (22 types):**

| Period | Period_Id | Description | Example (if today = 2025-01-18) |
|--------|-----------|-------------|----------------------------------|
| All | 1 | Entire date range (2000-2050) | 2000-01-01 to 2050-12-31 |
| Today | 1 | Current date only | 2025-01-18 |
| Yesterday | 2 | Previous day | 2025-01-17 |
| This Week | 3 | Current week (Mon-Sun) | 2025-01-13 to 2025-01-19 |
| Current Week to Date | 4 | Week start to today | 2025-01-13 to 2025-01-18 |
| Previous Week | 5 | Last complete week | 2025-01-06 to 2025-01-12 |
| Last 14 Days | 6 | Rolling 14 days | 2025-01-05 to 2025-01-18 |
| Last 30 Days | 6 | Rolling 30 days | 2024-12-20 to 2025-01-18 |
| Last 60 Days | 6 | Rolling 60 days | 2024-11-20 to 2025-01-18 |
| This Month | 7 | Current calendar month | 2025-01-01 to 2025-01-31 |
| Current Month to Date | 8 | Month start to today | 2025-01-01 to 2025-01-18 |
| Previous Month | 9 | Last complete month | 2024-12-01 to 2024-12-31 |
| Last 3 Month | 9 | Rolling 3 months | 2024-10-01 to 2025-01-18 |
| Last 6 Month | 9 | Rolling 6 months | 2024-07-01 to 2025-01-18 |
| Last 12 Month | 9 | Rolling 12 months | 2024-01-01 to 2025-01-18 |
| This Year | 10 | Current calendar year | 2025-01-01 to 2025-12-31 |
| This Fiscal Year | 10 | Current fiscal year (Jul-Jun) | 2024-07-01 to 2025-06-30 |
| Current Year To Date | 11 | Year start to today | 2025-01-01 to 2025-01-18 |
| Current Fiscal Year To Date | 11 | Fiscal year start to today | 2024-07-01 to 2025-01-18 |
| Previous Year | 12 | Last complete calendar year | 2024-01-01 to 2024-12-31 |
| Previous Fiscal Year | 12 | Last complete fiscal year | 2023-07-01 to 2024-06-30 |

---

## DAX Query Patterns

### Period definitions with date counts

```dax
EVALUATE
SUMMARIZECOLUMNS(
    Relative_Dates[Period],
    Relative_Dates[Period_Id],
    "Date Count", COUNTROWS(Relative_Dates),
    "Min Date", MIN(Relative_Dates[Date_Key]),
    "Max Date", MAX(Relative_Dates[Date_Key])
)
ORDER BY [Period_Id], [Period]
```

### Dates in "Last 30 Days" period

```dax
EVALUATE
ADDCOLUMNS(
    FILTER(
        Relative_Dates,
        [Period] = "Last 30 Days"
    ),
    "Date", RELATED(Dim_Date_Reference[Date]),
    "Day Name", RELATED(Dim_Date_Reference[Day_Name]),
    "Is Weekend", RELATED(Dim_Date_Reference[Day_of_Week]) IN {6, 7}
)
ORDER BY [Date_Key]
```

### Period overlap analysis

```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_Date_Reference[Date],
    "Period Count", COUNTROWS(RELATEDTABLE(Relative_Dates)),
    "Periods", CONCATENATEX(
        RELATEDTABLE(Relative_Dates),
        [Period],
        ", ",
        [Period_Id],
        ASC
    )
)
ORDER BY [Date]
```

### Current fiscal year measure

```dax
Assignments This Fiscal Year = 
CALCULATE(
    COUNTROWS(Assignments),
    Relative_Dates[Period] = "This Fiscal Year"
)
```

---

## Data Model Pattern

**Pattern:** Bridge Table (Many-to-Many with Bidirectional Filtering)

**Characteristics:**
- **Many-to-Many Relationships**: One date can belong to multiple periods; one period contains many dates
- **Bidirectional Cross-Filtering**: Enables Period → Date and Date → Period filtering
- **Dynamic Date Calculations**: Refreshes daily with current date-based relative periods
- **Timezone Awareness**: TodaysDate adjusted by tenant offset
- **Pre-Expanded Rows**: Each date within each period stored as individual rows

**Bridge Table Pattern:**
```
Dim_Relative_Dates_Type (1) → (Many) Relative_Dates (Many) ← (1) Dim_Date_Reference
                                           ↕ (bidirectional)
                                      Fact Tables
```

**How It Works:**
1. **Period Selection**: User selects "Last 30 Days" in slicer
2. **Bridge Filtering**: Filter flows through Relative_Dates to Dim_Date_Reference
3. **Bidirectional Flow**: Date filter flows back to fact tables via role-playing dimensions
4. **Result**: All fact records with dates in the last 30 days are displayed

**Bidirectional Cross-Filtering:**
Required to enable relative date filtering to flow through the date dimension to fact tables. Without bidirectional filtering, period selections would not affect fact table measures.

**Period_Id Grouping:**
Multiple periods can share the same Period_Id (e.g., Last 14 Days, Last 30 Days, Last 60 Days all have Period_Id = 6). This enables grouping related periods in visualizations.

---

## Related Documentation

### ERD Documents
- [ERD #2: Date Dimensions & Time Intelligence](../ERDs/ERD_02_Date_Dimensions_Time_Intelligence.md) - Bridge table architecture

### Related Tables
- **Dim_Date_Reference** - Master date dimension (linked via Date_Key)
- **Dim_Relative_Dates_Type** - Period definitions and metadata
- **Fn_Create_Dates** - Power Query function to expand date ranges
- **Dim_Tenant** - Provides timezone offset for TodaysDate calculation

### Other Documentation
- [ERD_Overview.md](../ERD_Overview.md) - Bridge table and bidirectional filtering patterns
- [Dim_Date_Reference.md](Dim_Date_Reference.md) - Master date dimension

---

## Change History

| Date | Change Description | Modified By |
|------|-------------------|-------------|
| 2025-11-18 | Initial table documentation created from TMDL metadata | AI Documentation Generator |

---

## Notes

**Refresh Frequency:** This table should be refreshed daily to keep "Today", "Yesterday", and rolling period calculations current. Each refresh recalculates relative dates based on the current date.

**Timezone Handling:** TodaysDate adjusted by tenant's offset minutes from Dim_Tenant[Offset Minutes], ensuring relative dates align with tenant's local timezone rather than UTC.

**Fiscal Year:** Hardcoded to July 1 - June 30. Organizations with different fiscal years need to modify fiscal year calculations in the Ranges list.

**Performance:** Bridge table with bidirectional filtering can impact performance on large datasets. Test performance with your data volumes and consider alternatives (e.g., DAX measures with DATESINPERIOD) if performance issues arise.

**Period_Id Reuse:** Multiple periods sharing the same Period_Id (Last 14/30/60 Days = 6, month-related periods = 9) enables grouping in visualizations. This is intentional design for organizing related periods.

**Week Start:** Weeks start on Monday (Day.Monday parameter in Date.StartOfWeek), consistent with ISO 8601 standard.

**"All" Period:** Includes entire 50-year date range (2000-2050), useful for "remove all filters" scenarios.

**Fn_Create_Dates Function:** Custom Power Query function (not shown in source) that expands date ranges into individual date rows with Period and Period_Id columns. This function is referenced but defined elsewhere in the model.
