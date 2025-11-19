# Assignments Table

**Last Updated**: November 18, 2025

## Overview

Contains all assignment records, including details such as assignment codes, status, assigned users, start and end dates, completion times, team information, and effort tracking. This table is the core source for analyzing work assignments, progress, and performance trends across teams and time.

## Table Specifications

| Property | Value |
|----------|-------|
| **Table Name** | Assignments |
| **Table Type** | Fact Table |
| **Lineage Tag** | c8c6dc1d-4eee-492d-b383-46a2e08e362c |
| **Refresh Policy** | Incremental Refresh (5 year rolling window, 1 year increments) |
| **Incremental Column** | Created_Date |
| **Source View** | VW_Assignments (Analytic Database) |
| **Related ERDs** | [ERD #1: Assignment Core](../ERDs/ERD_01_Assignment_Core.md), [ERD #4: Assignment Details](../ERDs/ERD_04_Assignment_Details_Snapshots.md) |

---

## Columns

### Source Columns (25)

| Column Name | Data Type | Description | Format | Aggregation |
|-------------|-----------|-------------|--------|-------------|
| **Tenant_Id** | string | Tenant identifier for the assignment | - | None |
| **Assignment_Id** | string | Unique identifier for the assignment | - | None |
| **Assignment_Code** | string | Business code or reference for the assignment | - | None |
| **AssignmentPoint_Id** | string | Identifier for the assignment point (location/asset) | - | None |
| **Assigned_To** | string | User or resource currently assigned to the assignment | - | None |
| **From_Date** | datetime | Start date/time of the assignment | dd/MM/yyyy HH:mm | None |
| **To_Date** | datetime | End date/time of the assignment | dd/MM/yyyy HH:mm | None |
| **Status_Id** | int64 | Numeric status identifier for the assignment | 0 | None |
| **Created_By** | string | Identifier of the user who created the assignment | - | None |
| **Created_Date** | datetime | Timestamp when the assignment was created | General Date | None |
| **Last_Updated** | datetime | Timestamp when the assignment was last updated | General Date | None |
| **WorkTemplate_Id** | string | Identifier of the work template used for the assignment | - | None |
| **Team_Id** | string | Identifier of the team responsible for the assignment | - | None |
| **Completed_By** | string | Identifier of the user who completed the assignment | - | None |
| **Finalised_By** | string | Identifier of the user who finalised the assignment | - | None |
| **Completed_On** | datetime | Timestamp when the assignment was completed | dd/MM/yyyy HH:mm | None |
| **Finalised_On** | datetime | Timestamp when the assignment was finalised | dd/MM/yyyy HH:mm | None |
| **Cancelled_On** | datetime | Timestamp when the assignment was cancelled | dd/MM/yyyy HH:mm | None |
| **Declined_On** | datetime | Timestamp when the assignment was declined | dd/MM/yyyy HH:mm | None |
| **Required_Date** | datetime | Date/time by which the assignment is required | dd/MM/yyyy HH:mm | None |
| **Assignment_Category_Id** | string | Identifier of the assignment category | - | None |
| **Assignment_Title** | string | Title or short description of the assignment | - | None |
| **Revision** | string | Revision identifier for the assignment record | - | None |
| **Effort_Mins** | int64 | Effort logged for the assignment in minutes | #,0 | Sum |
| **From_Date_Datekey** | int64 | Integer date key (YYYYMMDD) for the From_Date | 0 | None |
| **Completed_On_Datekey** | int64 | Integer date key (YYYYMMDD) for the Completed_On timestamp | 0 | Sum |
| **Finalised_On_Datekey** | int64 | Integer date key (YYYYMMDD) for the Finalised_On timestamp | 0 | None |

---

## Calculated Columns

### Field_Operators
**Description**: Comma-separated field operators associated with the assignment

**DAX Expression**:
```dax
LOOKUPVALUE(
    Lkp_Assignment_FieldOperators[Field_Operators],
    Lkp_Assignment_FieldOperators[Assignment_Id], 
    Assignments[Assignment_Id]
)
```

---

### URL
**Description**: Direct URL to open the assignment in the application

**Data Category**: WebUrl

**DAX Expression**:
```dax
MAX(Dim_Tenant[Tenant_URL]) & "/#/assignments/edit/" & 
Assignments[Assignment_Id] & "/modal"
```

---

### Completed_By_Full_Name
**Description**: Full name of the user who completed the assignment

**DAX Expression**:
```dax
RELATED(Dim_Users_Completed_By[Full_Name])
```

---

### Effort_Hrs
**Description**: Effort for the assignment expressed in hours (derived from minutes or duration)

**Format**: 0.0

**DAX Expression**:
```dax
VAR DurationHr = DATEDIFF(Assignments[From_Date], Assignments[To_Date], SECOND) / 3600
VAR Efforthrs = Assignments[Effort_Mins] / 60.0
RETURN
    IF(DurationHr < 24, COALESCE(Efforthrs, 1), Efforthrs)
```

**Business Logic**: For short assignments (<24 hours), defaults to logged effort or 1 hour if not logged. For longer assignments, uses logged effort only.

---

### Finalised_By_Full_Name
**Description**: Full name of the user who finalised the assignment

**DAX Expression**:
```dax
RELATED(Dim_Users_Finalised_By[Full_Name])
```

---

### Is_Outstanding
**Description**: Flag ("1"/"0") indicating whether the assignment is outstanding/past due

**DAX Expression**:
```dax
VAR Today = UTCNOW() + TIME(0, MAX(Dim_Tenant[Offset Minutes]), 0)
RETURN
    IF(Assignments[Status_Id] <> 7 && Assignments[To_Date] < Today, "1", "0")
```

**Business Logic**: Assignment is outstanding if not cancelled (Status 7) and To_Date is before current tenant time.

---

### Is_Assigned
**Description**: Flag ("1"/"0") indicating whether the assignment is currently assigned

**DAX Expression**:
```dax
IF(ISBLANK(Assignments[Assigned_To]), "0", "1")
```

---

### Is_Completed
**Description**: Flag ("1"/"0") indicating whether the assignment is completed

**DAX Expression**:
```dax
IF(Assignments[Status_Id] IN {3, 4, 7}, "1", "0")
```

**Business Logic**: Statuses 3, 4, and 7 indicate completion states.

---

### Initial_From_Date
**Description**: Initial From_Date captured from the first snapshot for the assignment

**Format**: dd/MM/yyyy HH:mm

**DAX Expression**:
```dax
RELATED(Assignment_FromDate_First_Snapshot[Initial_From_Date])
```

**Purpose**: Used to track if an assignment has been rescheduled from its original start date.

---

### From_Date_Hour
**Description**: Rounded numeric hour value derived from From_Date for reporting

**Format**: 0.00

**DAX Expression**:
```dax
VAR Timestamp = Assignments[From_Date]
VAR MinSecHour = ROUNDUP(MINUTE(Timestamp) / 60 + SECOND(Timestamp) / 3600, 2)
VAR MinSecHourRoundUp = SWITCH(
    TRUE(),
    MinSecHour > 0 && MinSecHour <= 0.25, 0.25,
    MinSecHour > 0.25 && MinSecHour <= 0.5, 0.5,
    MinSecHour > 0.5 && MinSecHour <= 0.75, 0.75,
    MinSecHour > 0.75 && MinSecHour <= 1, 1,
    MinSecHour
)
VAR Result = HOUR(Timestamp) + MinSecHourRoundUp
RETURN
    IF(Result = 24, 23.75, Result)
```

**Purpose**: Rounds time to 15-minute intervals (0.25 hour increments) for shift-based analysis.

---

### Shift_Date
**Description**: Date adjusted to shift boundaries based on From_Date

**Format**: Short Date

**DAX Expression**:
```dax
VAR FromDate = DATEVALUE(Assignments[From_Date])
VAR DateOffSet = LOOKUPVALUE(
    Dim_Shift_Time[DateOffset], 
    Dim_Shift_Time[Hour], 
    Assignments[From_Date_Hour]
)
RETURN
    FromDate + DateOffSet
```

**Purpose**: Aligns assignments to shift calendar (e.g., night shift work appears on the correct shift date).

---

### Field Operators & Assigned
**Description**: Display text combining field operators and assigned user where applicable

**DAX Expression**:
```dax
VAR AssignedTO = RELATED(Dim_Users_AssignedTo[Full_Name])
RETURN
    IF(
        AssignedTO = Assignments[Field_Operators] || ISBLANK(AssignedTO),
        Assignments[Field_Operators],
        AssignedTO & " - " & Assignments[Field_Operators]
    )
```

**Purpose**: Provides a consolidated view of all people involved in the assignment.

---

### Completed_On_Hour
**Description**: Rounded numeric hour value derived from Completed_On

**Format**: 0.00

**DAX Expression**:
```dax
VAR Timestamp = Assignments[Completed_On]
VAR MinSecHour = ROUNDUP(MINUTE(Timestamp) / 60 + SECOND(Timestamp) / 3600, 2)
VAR MinSecHourRoundUp = SWITCH(
    TRUE(),
    MinSecHour > 0 && MinSecHour <= 0.25, 0.25,
    MinSecHour > 0.25 && MinSecHour <= 0.5, 0.5,
    MinSecHour > 0.5 && MinSecHour <= 0.75, 0.75,
    MinSecHour > 0.75 && MinSecHour <= 1, 1,
    MinSecHour
)
VAR Result = HOUR(Timestamp) + MinSecHourRoundUp
RETURN
    IF(Result = 24, 23.75, Result)
```

---

### Completed_On_ShiftDate_Datekey
**Description**: Integer date key (YYYYMMDD) for Completed_On adjusted to shift date logic

**Data Type**: int64

**DAX Expression**:
```dax
VAR HourofDay = HOUR(Assignments[Completed_On])
VAR DayOfWeek = WEEKDAY(Assignments[Completed_On]) // 2 is Monday
VAR ShiftDate = IF(
    DayOfWeek = 2 && HourofDay < 6,
    Assignments[Completed_On] - 1,
    Assignments[Completed_On]
)
RETURN
    IF(NOT(ISBLANK(ShiftDate)), FORMAT(ShiftDate, "YYYYMMDD"))
```

**Purpose**: Adjusts Monday early morning completions to previous day's shift.

---

## Relationships

### Active Relationships

| To Table | From Column | To Column | Cardinality | Description |
|----------|-------------|-----------|-------------|-------------|
| **Dim_Teams** | Team_Id | Team_Id | Many-to-One | Team responsible for assignment |
| **Dim_Assignment_Categories** | Assignment_Category_Id | Id | Many-to-One | Assignment classification |
| **Dim_Users_AssignedTo** | Assigned_To | User_Id | Many-to-One | User assigned to perform the work |
| **Dim_WorkTemplates** | WorkTemplate_Id | Id | Many-to-One | Template defining assignment structure |
| **Dim_Assignment_Status** | Status_Id | Id | Many-to-One | Current status lookup |
| **Dim_Date_FromDate** | From_Date_Datekey | Date_Key | Many-to-One | Start date dimension |
| **Dim_Date_FinalisedOn** | Finalised_On_Datekey | Date_Key | Many-to-One | Finalisation date dimension |
| **Dim_AssignmentPoints** | AssignmentPoint_Id | Id | Many-to-One | Location/asset hierarchy |
| **Dim_Users_Completed_By** | Completed_By | User_Id | Many-to-One | User who completed the work |
| **Dim_Users_Finalised_By** | Finalised_By | User_Id | Many-to-One | User who finalised the assignment |
| **Dim_Is_Outstanding** | Is_Outstanding | Value | Many-to-One | Outstanding flag lookup |
| **Dim_Is_Completed** | Is_Completed | Value | Many-to-One | Completed flag lookup |
| **Dim_Is_Assigned** | Is_Assigned | Value | Many-to-One | Assigned flag lookup |
| **Dim_Shift_Time** | From_Date_Hour | Hour | Many-to-One | Shift time analysis |
| **Dim_Date_ShiftDate** | Shift_Date | Date | Many-to-One | Shift-adjusted date |

### Inactive Relationships

| To Table | From Column | To Column | Purpose |
|----------|-------------|-----------|---------|
| **Dim_Users_AssignedTo** | Created_By | User_Id | Can be activated for creator analysis |
| **Dim_Date_CompletedOn** | Completed_On_Datekey | Date_Key | Use USERELATIONSHIP for completion date analysis |
| **Dim_Shift_Time** | Completed_On_Hour | Hour | Use USERELATIONSHIP for completion shift analysis |
| **Dim_Date_CompletedOn** | Completed_On_ShiftDate_Datekey | Date_Key | Use USERELATIONSHIP for shift-adjusted completion |

### Bridge Tables (Many-to-Many via)

| Bridge Table | Purpose |
|--------------|---------|
| **Assignment_FieldOperators** | Links assignments to multiple field operators (bidirectional) |
| **Assignment_Tags** | Key-value tags for assignments |
| **Assignment_Details_Snapshot** | Historical state tracking |
| **Assignment_Completion_Percentage_Snapshot** | Progress tracking (1:1) |
| **Assignment_Declined_Reasons** | Decline/hold reasons (1:1) |
| **Assignment_FromDate_First_Snapshot** | Original start date tracking (1:1) |

---

## Common DAX Query Patterns

### Count Assignments by Status
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_Assignment_Status[Status_Name],
    "Assignment Count", DISTINCTCOUNT(Assignments[Assignment_Id]),
    "Total Effort Hours", SUM(Assignments[Effort_Hrs])
)
ORDER BY [Assignment Count] DESC
```

### Assignments Overdue
```dax
EVALUATE
FILTER(
    ADDCOLUMNS(
        Assignments,
        "Days Overdue", INT(UTCNOW() - Assignments[To_Date])
    ),
    Assignments[Is_Outstanding] = "1"
)
ORDER BY Assignments[To_Date]
```

### Assignment Completion Rate by Team
```dax
EVALUATE
ADDCOLUMNS(
    VALUES(Dim_Teams[Name]),
    "Total Assignments", CALCULATE(COUNTROWS(Assignments)),
    "Completed Assignments", 
        CALCULATE(
            COUNTROWS(Assignments),
            Assignments[Is_Completed] = "1"
        ),
    "Completion Rate", 
        DIVIDE(
            CALCULATE(COUNTROWS(Assignments), Assignments[Is_Completed] = "1"),
            CALCULATE(COUNTROWS(Assignments)),
            0
        )
)
ORDER BY [Completion Rate] DESC
```

### Assignments Rescheduled
```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Assignments,
        "Assignment Code", Assignments[Assignment_Code],
        "Initial Start", Assignments[Initial_From_Date],
        "Current Start", Assignments[From_Date],
        "Days Shifted", DATEDIFF(
            Assignments[Initial_From_Date], 
            Assignments[From_Date], 
            DAY
        )
    ),
    NOT(ISBLANK(Assignments[Initial_From_Date])) &&
    Assignments[From_Date] <> Assignments[Initial_From_Date]
)
ORDER BY [Days Shifted] DESC
```

### Average Assignment Duration by Template
```dax
EVALUATE
ADDCOLUMNS(
    VALUES(Dim_WorkTemplates[Name]),
    "Avg Duration Hours", 
        AVERAGEX(
            RELATEDTABLE(Assignments),
            DATEDIFF(
                Assignments[From_Date],
                Assignments[To_Date],
                HOUR
            )
        ),
    "Avg Effort Hours",
        AVERAGE(Assignments[Effort_Hrs]),
    "Assignment Count",
        COUNTROWS(RELATEDTABLE(Assignments))
)
ORDER BY [Assignment Count] DESC
```

---

## Related Documentation

- **[ERD #1: Assignment Core Model](../ERDs/ERD_01_Assignment_Core.md)** - Core assignment entities and relationships
- **[ERD #2: Date Dimensions](../ERDs/ERD_02_Date_Dimensions_Time_Intelligence.md)** - Date relationship patterns
- **[ERD #4: Assignment Details & Snapshots](../ERDs/ERD_04_Assignment_Details_Snapshots.md)** - Historical tracking
- **[Dim_AssignmentPoints](Dim_AssignmentPoints.md)** - Location hierarchy
- **[Dim_WorkTemplates](Dim_WorkTemplates.md)** - Template definitions
- **[TimeSeries](TimeSeries.md)** - Field measurement data capture

---

## Change History

| Date | Change | Author |
|------|--------|--------|
| 2025-11-18 | Initial documentation generated from TMDL | AI Documentation Generator |

