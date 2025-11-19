# Dim_Date_Reference

**Table Type:** Dimension Table (Calculated - Role-Playing)  
**Source:** Dim_Date_FromDate  
**Primary Key:** Date  
**Related ERD:** [ERD #2: Date Dimensions & Time Intelligence](../ERDs/ERD_02_Date_Dimensions_Time_Intelligence.md)

---

## Table Overview

Master date dimension providing comprehensive calendar attributes for date-based analysis. Contains one row per day from 2000-01-01 to 2050-12-31 (18,262 days) with detailed date hierarchies including calendar year/quarter/month/week/day, fiscal year attributes, and date intelligence helpers. This table serves as the source for six role-playing date dimensions used throughout the semantic model.

**Source System:** Calculated table from Dim_Date_FromDate

**Row Count:** 18,262 rows (50 years: 2000-2050)

**Refresh Type:** Calculated table (no refresh needed)

**Multi-Tenant:** No (shared reference dimension across all tenants)

**Role-Playing Contexts:** Source for 6 date dimensions with different relationship contexts

---

## Table Specifications

| Property | Value |
|----------|-------|
| Lineage Tag | ac7537de-7286-4db7-80e6-0a1c2fd7a3a8 |
| Query Group | None (calculated table) |
| Partitions | 1 (Dim_Date_Reference - calculated) |
| Relationships | Multiple outbound to Relative_Dates bridge table |
| Calculated Columns | 0 (all columns inherited from source) |
| Hierarchies | 0 (hierarchies defined in role-playing dimensions) |

---

## Column Specifications

| Column Name | Data Type | Format | Description | Sorted By | Nullable |
|-------------|-----------|--------|-------------|-----------|----------|
| **Date** | date | dd/mm/yyyy | Calendar date value | - | No |
| **Year** | int64 | 0 | Calendar year number (2000-2050) | - | No |
| **Year_Start** | date | dd/mm/yyyy | First day of the calendar year | - | No |
| **Month** | int64 | 0 | Month number (1-12) | - | No |
| **Month_Start** | date | MMM-yy | First day of the month | - | No |
| **Days_in_Month** | int64 | 0 | Number of days in the month (28-31) | - | No |
| **Day_of_Month** | int64 | 0 | Day number within month (1-31) | - | No |
| **Day_Name** | string | - | Day name (Monday, Tuesday, etc.) | Day_of_Week | No |
| **Day_of_Week** | int64 | 0 | Weekday index (1=Monday, 7=Sunday) | - | No |
| **Day_of_Year** | int64 | 0 | Day number within year (1-366) | - | No |
| **Month_Name** | string | - | Month name (January, February, etc.) | - | No |
| **Quarter** | int64 | 0 | Calendar quarter number (1-4) | - | No |
| **Quarter_Start** | date | MMM-yy | First day of the quarter | - | No |
| **Week_of_Year** | int64 | 0 | Week number within year (1-53) | - | No |
| **Week_of_Month** | int64 | 0 | Week number within month (1-5) | - | No |
| **Week_Start** | date | dd/mm/yyyy | First day of the week (Monday) | - | No |
| **Week_End** | date | dd/mm/yyyy | Last day of the week (Sunday) | - | No |
| **Fiscal_Year** | int64 | 0 | Fiscal year (July 1 - June 30) | - | No |
| **Fiscal_Quarter** | int64 | 0 | Fiscal quarter number (1-4) | - | No |
| **Fiscal_Month** | int64 | 0 | Fiscal month number (1-12) | - | No |
| **Month_Year_Number** | int64 | 0 | Numeric month-year key (YYYYMM) | - | No |
| **Month_Year_Display** | string | - | Month-year display label (Jan-2025) | Month_Year_Number | No |
| **Date_Key** | int64 | 0 | Integer date key (YYYYMMDD) | - | No |
| **Is_Today** | boolean | - | Flag indicating current date | - | No |

**Note:** Day_Name sorted by Day_of_Week ensures proper weekday ordering (Monday → Sunday). Month_Year_Display sorted by Month_Year_Number ensures chronological ordering.

---

## Calculated Columns

This table has no calculated columns. All columns are inherited from the source table Dim_Date_FromDate via the DAX calculated table expression.

---

## Relationships

### Role-Playing Date Dimensions

This table serves as the source for six role-playing date dimensions that provide different relationship contexts:

1. **Dim_Date_Assignment_Created_Date** - Assignment creation dates
2. **Dim_Date_Assignment_Completed_Date** - Assignment completion dates  
3. **Dim_Date_Assignment_Due_Date** - Assignment due dates
4. **Dim_Date_Assignment_Updated_Date** - Assignment last update dates
5. **Dim_Date_Assignment_Started_Date** - Assignment start dates
6. **Dim_Date_Assignment_Finalised_Date** - Assignment finalized dates

Each role-playing dimension contains the same columns and data but maintains separate relationships to the Assignments fact table.

### Outbound Relationships

**To Relative_Dates (Bridge Table)**
- **Type:** One-to-many (via Date_Key)
- **From Column:** Date_Key
- **To Column:** Relative_Dates[Date_Key]
- **Cardinality:** 1:Many
- **Cross-filter Direction:** Both (bidirectional)
- **Purpose:** Enables relative date filtering (Today, Last 30 Days, This Month, etc.)

---

## DAX Source

```dax
Dim_Date_Reference = Dim_Date_FromDate
```

**Logic:** Simple calculated table that references Dim_Date_FromDate, creating a copy of all rows and columns for use as a reference dimension.

**Purpose:** Provides a clean reference dimension separate from the role-playing dimensions used in active relationships.

---

## DAX Query Patterns

### Date range overview

```dax
EVALUATE
SUMMARIZECOLUMNS(
    "Min Date", MIN(Dim_Date_Reference[Date]),
    "Max Date", MAX(Dim_Date_Reference[Date]),
    "Total Days", COUNTROWS(Dim_Date_Reference),
    "Years Covered", DISTINCTCOUNT(Dim_Date_Reference[Year])
)
```

### Fiscal year calendar

```dax
EVALUATE
ADDCOLUMNS(
    FILTER(
        Dim_Date_Reference,
        [Fiscal_Year] = 2025
    ),
    "Date", [Date],
    "FY", [Fiscal_Year],
    "FQ", [Fiscal_Quarter],
    "FM", [Fiscal_Month],
    "Calendar Year", [Year],
    "Calendar Quarter", [Quarter]
)
ORDER BY [Date]
```

### Week boundaries by month

```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_Date_Reference[Year],
    Dim_Date_Reference[Month_Name],
    Dim_Date_Reference[Week_of_Month],
    "Week Start", MIN(Dim_Date_Reference[Week_Start]),
    "Week End", MAX(Dim_Date_Reference[Week_End]),
    "Days", COUNTROWS(Dim_Date_Reference)
)
ORDER BY [Year], [Month_Name], [Week_of_Month]
```

### Today flag validation

```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Dim_Date_Reference,
        "Date", [Date],
        "Is Today", [Is_Today],
        "Day Name", [Day_Name],
        "Days from Today", DATEDIFF([Date], TODAY(), DAY)
    ),
    [Is Today] = TRUE()
)
```

### Date key format validation

```dax
EVALUATE
ADDCOLUMNS(
    TOPN(10, Dim_Date_Reference, [Date], ASC),
    "Date", [Date],
    "Date_Key", [Date_Key],
    "Reconstructed Date", 
        DATE(
            QUOTIENT([Date_Key], 10000),
            QUOTIENT(MOD([Date_Key], 10000), 100),
            MOD([Date_Key], 100)
        ),
    "Match", [Date] = DATE(
        QUOTIENT([Date_Key], 10000),
        QUOTIENT(MOD([Date_Key], 10000), 100),
        MOD([Date_Key], 100)
    )
)
```

### Fiscal vs calendar year comparison

```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_Date_Reference[Year],
    "Calendar Q1", CALCULATE(COUNTROWS(Dim_Date_Reference), [Quarter] = 1),
    "Calendar Q2", CALCULATE(COUNTROWS(Dim_Date_Reference), [Quarter] = 2),
    "Calendar Q3", CALCULATE(COUNTROWS(Dim_Date_Reference), [Quarter] = 3),
    "Calendar Q4", CALCULATE(COUNTROWS(Dim_Date_Reference), [Quarter] = 4),
    "Fiscal Q1 (Jul-Sep)", CALCULATE(COUNTROWS(Dim_Date_Reference), [Fiscal_Quarter] = 1),
    "Fiscal Q2 (Oct-Dec)", CALCULATE(COUNTROWS(Dim_Date_Reference), [Fiscal_Quarter] = 2),
    "Fiscal Q3 (Jan-Mar)", CALCULATE(COUNTROWS(Dim_Date_Reference), [Fiscal_Quarter] = 3),
    "Fiscal Q4 (Apr-Jun)", CALCULATE(COUNTROWS(Dim_Date_Reference), [Fiscal_Quarter] = 4)
)
ORDER BY [Year]
```

---

## Data Model Pattern

**Pattern:** Role-Playing Dimension (Reference Source)

**Characteristics:**
- **Master Date Dimension**: Single source of truth for date attributes
- **50-Year Range**: 2000-2050 coverage (18,262 days)
- **Comprehensive Attributes**: Calendar, fiscal, week, day, and display attributes
- **Date_Key Standard**: YYYYMMDD integer format for efficient joins
- **Role-Playing Source**: Copied by 6 role-playing dimensions with different relationship contexts
- **Sorted Columns**: Day_Name and Month_Year_Display sorted for proper ordering

**Fiscal Year Configuration:**
- **Fiscal Year**: July 1 - June 30
- **Fiscal Quarter 1**: July - September
- **Fiscal Quarter 2**: October - December
- **Fiscal Quarter 3**: January - March
- **Fiscal Quarter 4**: April - June

**Role-Playing Dimension Pattern:**
The semantic model uses role-playing dimensions to enable multiple date contexts in a single fact table:

```
Dim_Date_Reference (Reference)
    ↓ (source)
    ├─ Dim_Date_Assignment_Created_Date → Assignments[Created_Date_Key]
    ├─ Dim_Date_Assignment_Completed_Date → Assignments[Completed_Date_Key]
    ├─ Dim_Date_Assignment_Due_Date → Assignments[Due_Date_Key]
    ├─ Dim_Date_Assignment_Updated_Date → Assignments[Updated_Date_Key]
    ├─ Dim_Date_Assignment_Started_Date → Assignments[Started_Date_Key]
    └─ Dim_Date_Assignment_Finalised_Date → Assignments[Finalised_Date_Key]
```

Each role-playing dimension enables independent filtering and time intelligence:
- Created Date: When was the assignment created?
- Due Date: When is the assignment due?
- Completed Date: When was the assignment completed?
- Updated Date: When was the assignment last modified?
- Started Date: When did work begin?
- Finalised Date: When was the assignment locked/approved?

**USERELATIONSHIP Pattern:**
Measures can switch between date contexts using USERELATIONSHIP:

```dax
Assignments by Due Date = 
CALCULATE(
    COUNTROWS(Assignments),
    USERELATIONSHIP(Assignments[Due_Date_Key], Dim_Date_Assignment_Due_Date[Date_Key])
)
```

---

## Related Documentation

### ERD Documents
- [ERD #2: Date Dimensions & Time Intelligence](../ERDs/ERD_02_Date_Dimensions_Time_Intelligence.md) - Complete date dimension architecture

### Related Tables
- **Dim_Date_FromDate** - Source table for this reference dimension
- **Dim_Date_Assignment_Created_Date** - Role-playing dimension for created dates
- **Dim_Date_Assignment_Completed_Date** - Role-playing dimension for completed dates
- **Dim_Date_Assignment_Due_Date** - Role-playing dimension for due dates
- **Dim_Date_Assignment_Updated_Date** - Role-playing dimension for updated dates
- **Dim_Date_Assignment_Started_Date** - Role-playing dimension for started dates
- **Dim_Date_Assignment_Finalised_Date** - Role-playing dimension for finalised dates
- **Relative_Dates** - Bridge table for relative date filtering
- **Dim_Relative_Dates_Type** - Relative period definitions

### Other Documentation
- [ERD_Overview.md](../ERD_Overview.md) - Role-playing dimension pattern documentation
- [Relative_Dates.md](Relative_Dates.md) - Bridge table for relative date filtering

---

## Change History

| Date | Change Description | Modified By |
|------|-------------------|-------------|
| 2025-11-18 | Initial table documentation created from TMDL metadata | AI Documentation Generator |

---

## Notes

**Date Range:** 2000-2050 provides 50 years of date coverage, sufficient for historical analysis and forward planning.

**Date_Key Format:** YYYYMMDD integer format (e.g., 20250118 for January 18, 2025) enables efficient integer joins and sorting. This format is used consistently across all fact tables with date foreign keys.

**Fiscal Year:** Hardcoded to July 1 - June 30. Organizations with different fiscal years may need to modify Fiscal_Year, Fiscal_Quarter, and Fiscal_Month calculations in the source table.

**Is_Today Flag:** Dynamically updated to identify the current date, useful for "as of today" filters and current date highlighting.

**Week Start:** Weeks start on Monday (Day_of_Week = 1) following ISO 8601 standard.

**Role-Playing Design:** Creating a separate reference table (vs using one of the role-playing dimensions directly) provides a clean separation between the master date data and the relationship contexts, improving model maintainability.

**Sorting:** Day_Name sorted by Day_of_Week and Month_Year_Display sorted by Month_Year_Number ensures proper ordering in slicers and visuals without custom sort columns.
