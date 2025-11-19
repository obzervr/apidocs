# Fact_Rosters

## Table Overview
`Fact_Rosters` is a fact table that stores scheduled roster assignments, tracking which users are assigned to which shifts and operating areas. Each row represents a planned roster entry with attendance details, shift times, position information, and SAP integration fields.

This table enables shift scheduling analysis, planned attendance tracking, and workforce capacity planning.

**Current Status**: Standard import table with soft delete filtering (IsDeleted = false).

---

## Specifications
- **Source**: `FactRosters` table
- **Row Count**: Moderate to high volume (roster entries × scheduling period)
- **Grain**: One row per user per shift assignment
- **Primary Key**: `Id`
- **Incremental Refresh**: Not enabled
- **Partitioning Strategy**: Standard import
- **Source Columns**: 26
- **Calculated Columns**: 2 (Planned_Shift_Hour, Shift_Hour)
- **Filtering**: IsDeleted = false (soft delete filtering)

---

## Column Specifications

| Column Name | Data Type | Format | Nullable | Hidden | Description |
|------------|-----------|--------|----------|--------|-------------|
| Id | Int64 | | No | No | Primary key for roster entry |
| Operating_Area | String | | Yes | No | Operating area or location for the shift |
| User_Id | Int64 | | No | No | Foreign key to user dimension |
| Shift_Id | Int64 | | Yes | No | Foreign key to shift dimension |
| Personnel_No | String | | Yes | No | Personnel number from HR/payroll system |
| First_Name | String | | Yes | No | User's first name |
| Surname | String | | Yes | No | User's last name |
| Second_Name | String | | Yes | No | User's middle name |
| Planned_Attendance | Double | | Yes | No | Planned attendance hours/percentage |
| Planned_Absence | Double | | Yes | No | Planned absence hours/percentage |
| Attendance_Name | String | | Yes | No | Attendance type description |
| Attendance_Type | String | | Yes | No | Attendance type classification code |
| Attendance_Reference | String | | Yes | No | Reference code for attendance type |
| Planned_Start_Datetime | DateTime | yyyy-MM-dd HH:mm:ss | Yes | No | Scheduled shift start time |
| Planned_End_Datetime | DateTime | yyyy-MM-dd HH:mm:ss | Yes | No | Scheduled shift end time |
| Position_Description | String | | Yes | No | Job position or role description |
| Process_Group | String | | Yes | No | Process or work group classification |
| SAP_WSR_Description | String | | Yes | No | SAP Work Schedule Rule description |
| Personnel_Area | String | | Yes | No | Personnel area classification |
| SAP_Work_SCHEDULERule | String | | Yes | No | SAP Work Schedule Rule code |
| Tenant_Id | Int64 | | No | No | Foreign key to tenant dimension |
| Last_Updated | DateTime | yyyy-MM-dd HH:mm:ss | Yes | No | Timestamp of last modification |
| Created_Date | DateTime | yyyy-MM-dd HH:mm:ss | No | No | Creation timestamp |
| Point_Code | String | | Yes | No | Point or location code |

---

## Calculated Columns

### Planned_Shift_Hour
Calculates the planned shift duration in hours based on `Planned_Start_Datetime` and `Planned_End_Datetime`. Defaults to 12 hours if either datetime is missing.

```dax
Planned_Shift_Hour = 
VAR ShiftMinutes = 
    DATEDIFF(
        Fact_Rosters[Planned_Start_Datetime],
        Fact_Rosters[Planned_End_Datetime],
        MINUTE
    )
RETURN
    IF(
        ISBLANK(Fact_Rosters[Planned_Start_Datetime]) 
        || ISBLANK(Fact_Rosters[Planned_End_Datetime]),
        12,
        ShiftMinutes / 60
    )
```

### Shift_Hour
Retrieves the standard shift duration in hours from the `Dim_Shifts` dimension via relationship.

```dax
Shift_Hour = RELATED(Dim_Shifts[Shift_Hour])
```

---

## Relationships

### Outbound Relationships
| To Table | From Column(s) | To Column(s) | Cardinality | Cross Filter | Relationship ID |
|----------|---------------|--------------|-------------|--------------|-----------------|
| `Dim_Tenant` | Tenant_Id | Tenant_Id | Many-to-One | Single | (Tenant context) |
| `Dim_Users_AssignedTo` | User_Id | User_Id | Many-to-One | Single | (User context) |
| `Dim_Shifts` | Shift_Id | Shift_Id | Many-to-One | Single | (Shift context) |

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
            Operating_Area,
            User_Id,
            Shift_Id,
            Personnel_No,
            First_Name,
            Surname,
            Second_Name,
            Planned_Attendance,
            Planned_Absence,
            Attendance_Name,
            Attendance_Type,
            Attendance_Reference,
            Planned_Start_Datetime,
            Planned_End_Datetime,
            Position_Description,
            Process_Group,
            SAP_WSR_Description,
            Personnel_Area,
            SAP_Work_SCHEDULERule,
            Tenant_Id,
            Last_Updated,
            Created_Date,
            Point_Code
        FROM [dbo].[FactRosters]
        WHERE IsDeleted = 0"
    ),
    #"Filtered Tenant" = Table.SelectRows(
        Source,
        each [Tenant_Id] = TenantId
    ),
    #"Changed Types" = Table.TransformColumnTypes(
        #"Filtered Tenant",
        {
            {"Id", Int64.Type},
            {"Operating_Area", type text},
            {"User_Id", Int64.Type},
            {"Shift_Id", Int64.Type},
            {"Personnel_No", type text},
            {"First_Name", type text},
            {"Surname", type text},
            {"Second_Name", type text},
            {"Planned_Attendance", type number},
            {"Planned_Absence", type number},
            {"Attendance_Name", type text},
            {"Attendance_Type", type text},
            {"Attendance_Reference", type text},
            {"Planned_Start_Datetime", type datetime},
            {"Planned_End_Datetime", type datetime},
            {"Position_Description", type text},
            {"Process_Group", type text},
            {"SAP_WSR_Description", type text},
            {"Personnel_Area", type text},
            {"SAP_Work_SCHEDULERule", type text},
            {"Tenant_Id", Int64.Type},
            {"Last_Updated", type datetime},
            {"Created_Date", type datetime},
            {"Point_Code", type text}
        }
    )
in
    #"Changed Types"
```

---

## DAX Query Patterns

### Example 1: Shift Coverage by Operating Area
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Fact_Rosters[Operating_Area],
    Dim_Shifts[Shift_Name],
    "Rostered_Staff", DISTINCTCOUNT(Fact_Rosters[User_Id]),
    "Total_Roster_Entries", COUNTROWS(Fact_Rosters),
    "Total_Planned_Hours", SUM(Fact_Rosters[Planned_Shift_Hour])
)
ORDER BY Fact_Rosters[Operating_Area], Dim_Shifts[Shift_Name]
```

### Example 2: User Roster Schedule
```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Fact_Rosters,
        "User_Name", Fact_Rosters[First_Name] & " " & Fact_Rosters[Surname],
        "Shift", RELATED(Dim_Shifts[Shift_Name]),
        "Operating_Area", Fact_Rosters[Operating_Area],
        "Start", FORMAT(Fact_Rosters[Planned_Start_Datetime], "yyyy-MM-dd HH:mm"),
        "End", FORMAT(Fact_Rosters[Planned_End_Datetime], "yyyy-MM-dd HH:mm"),
        "Shift_Hours", Fact_Rosters[Planned_Shift_Hour],
        "Position", Fact_Rosters[Position_Description]
    ),
    Fact_Rosters[User_Id] = 42
)
ORDER BY Fact_Rosters[Planned_Start_Datetime] DESC
```

### Example 3: Attendance vs Absence Analysis
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Fact_Rosters[Attendance_Name],
    Fact_Rosters[Attendance_Type],
    "Entry_Count", COUNTROWS(Fact_Rosters),
    "Total_Planned_Attendance", SUM(Fact_Rosters[Planned_Attendance]),
    "Total_Planned_Absence", SUM(Fact_Rosters[Planned_Absence]),
    "Unique_Users", DISTINCTCOUNT(Fact_Rosters[User_Id])
)
ORDER BY [Entry_Count] DESC
```

### Example 4: Shift Roster Capacity
```dax
EVALUATE
ADDCOLUMNS(
    SUMMARIZE(
        Fact_Rosters,
        Dim_Shifts[Shift_Name],
        Dim_Shifts[Shift_Hour]
    ),
    "Rostered_Staff", COUNTROWS(Fact_Rosters),
    "Total_Shift_Hours_Capacity", SUM(Fact_Rosters[Planned_Shift_Hour]),
    "Avg_Hours_Per_Person", AVERAGE(Fact_Rosters[Planned_Shift_Hour]),
    "Operating_Areas_Covered", DISTINCTCOUNT(Fact_Rosters[Operating_Area])
)
ORDER BY Dim_Shifts[Shift_Name]
```

### Example 5: Position and Process Group Distribution
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Fact_Rosters[Position_Description],
    Fact_Rosters[Process_Group],
    "Roster_Count", COUNTROWS(Fact_Rosters),
    "Unique_Personnel", DISTINCTCOUNT(Fact_Rosters[Personnel_No]),
    "Avg_Planned_Hours", AVERAGE(Fact_Rosters[Planned_Shift_Hour]),
    "Operating_Areas", CONCATENATEX(
        VALUES(Fact_Rosters[Operating_Area]),
        Fact_Rosters[Operating_Area],
        ", "
    )
)
ORDER BY [Roster_Count] DESC
```

### Example 6: SAP Integration Fields Report
```dax
EVALUATE
SELECTCOLUMNS(
    TOPN(
        100,
        Fact_Rosters,
        Fact_Rosters[Created_Date],
        DESC
    ),
    "Personnel_No", Fact_Rosters[Personnel_No],
    "Name", Fact_Rosters[First_Name] & " " & Fact_Rosters[Surname],
    "SAP_WSR_Desc", Fact_Rosters[SAP_WSR_Description],
    "SAP_Work_Schedule_Rule", Fact_Rosters[SAP_Work_SCHEDULERule],
    "Personnel_Area", Fact_Rosters[Personnel_Area],
    "Shift", RELATED(Dim_Shifts[Shift_Name]),
    "Created", FORMAT(Fact_Rosters[Created_Date], "yyyy-MM-dd")
)
```

---

## Data Model Pattern

### Roster Scheduling Fact Pattern
`Fact_Rosters` implements a roster/scheduling fact pattern that stores planned workforce assignments to shifts and operating areas. This pattern bridges HR systems (SAP), shift management, and operational planning.

**Roster Entry Characteristics**:
- **Planned vs Actual**: This table stores *planned* rosters (scheduled shifts), not actual attendance
- **User Assignment**: Links users to specific shifts and operating areas
- **Time-Bound**: Each entry has planned start/end datetimes
- **Attendance Classification**: Planned attendance/absence tracking
- **SAP Integration**: Multiple SAP-specific fields for ERP integration

**Planned Attendance vs Absence**:
The dual tracking of `Planned_Attendance` and `Planned_Absence` enables:
- **Planned_Attendance**: Expected work hours or attendance percentage
- **Planned_Absence**: Scheduled time off, breaks, or non-working periods
- Together these define expected availability within the rostered period

**Example Values**:
- Planned_Attendance = 12.0, Planned_Absence = 0.0 (full 12-hour shift)
- Planned_Attendance = 10.5, Planned_Absence = 1.5 (12-hour shift with 1.5hr planned break)
- Planned_Attendance = 0.0, Planned_Absence = 8.0 (scheduled day off)

**Shift Hour Calculations**:
Two calculated columns provide shift duration:
1. **Planned_Shift_Hour**: Calculated from Planned_Start_Datetime and Planned_End_Datetime
   - Uses actual planned times (may vary from standard shift duration)
   - Defaults to 12 hours if datetimes are missing
   - Calculated in minutes then divided by 60 for precision

2. **Shift_Hour**: Retrieved from Dim_Shifts dimension
   - Standard shift duration definition
   - Enables comparison: planned vs standard duration
   - Identifies roster entries with non-standard hours

**SAP Integration Fields**:
Multiple SAP-specific columns enable ERP integration:
- **Personnel_No**: SAP HR personnel number (unique employee identifier)
- **SAP_WSR_Description**: Work Schedule Rule description (readable name)
- **SAP_Work_SCHEDULERule**: Work Schedule Rule code (technical identifier)
- **Personnel_Area**: SAP organizational unit or personnel area code

These fields enable:
- Bi-directional sync between Obzervr and SAP
- Roster reconciliation with HR systems
- Payroll integration
- Compliance reporting

**Operating Area Classification**:
`Operating_Area` enables location-based roster management:
- Assigns staff to specific operational zones (e.g., "Pit 3", "Processing Plant", "Workshop A")
- Enables area-specific capacity planning
- Supports multi-site operations
- May align with Work_Centre in team definitions

**Position and Process Group**:
Organizational classification fields:
- **Position_Description**: Job title or role (e.g., "Heavy Equipment Operator", "Maintenance Technician")
- **Process_Group**: Work process or functional group (e.g., "Haulage", "Maintenance", "Processing")

These enable:
- Skill-based roster analysis
- Competency coverage reporting
- Process-specific capacity planning

**Attendance Classification**:
Three attendance fields provide classification:
- **Attendance_Name**: Human-readable description (e.g., "Regular Shift", "Overtime", "Planned Leave")
- **Attendance_Type**: Type code or category
- **Attendance_Reference**: Reference code for integration or payroll

**Soft Delete Pattern**:
The WHERE clause filters `IsDeleted = 0`, implementing soft deletes:
- Roster entries are not permanently deleted
- IsDeleted flag marks cancelled or superseded rosters
- Historical roster data retained for audit
- Only active rosters appear in reports

**Point_Code Field**:
The `Point_Code` column may link to:
- Assignment points (locations/equipment)
- Time clock locations
- Geographic reference points
- Shift handover locations

**Example Scenario - 24/7 Mine Site Roster**:

**Day Shift Coverage** (7am-7pm):
| User_Name | Operating_Area | Position | Planned_Hours | Attendance |
|-----------|---------------|----------|---------------|------------|
| John Smith | Pit 3 | Heavy Equipment Operator | 12.0 | Regular Shift |
| Mary Jones | Pit 3 | Heavy Equipment Operator | 12.0 | Regular Shift |
| Bob Wilson | Pit 3 | Supervisor | 10.5 | Regular Shift |
| Sue Davis | Processing Plant | Plant Operator | 12.0 | Regular Shift |
| Mike Brown | Workshop A | Maintenance Technician | 8.0 | Partial Shift |
| Tom Lee | N/A | Maintenance Technician | 0.0 | Planned Leave (Planned_Absence=12.0) |

**Night Shift Coverage** (7pm-7am):
| User_Name | Operating_Area | Position | Planned_Hours | Attendance |
|-----------|---------------|----------|---------------|------------|
| Paul Green | Pit 3 | Heavy Equipment Operator | 12.0 | Regular Shift |
| Lisa White | Pit 3 | Heavy Equipment Operator | 12.0 | Regular Shift |
| Jim Black | Pit 3 | Supervisor | 12.0 | Regular Shift |
| Amy Gray | Processing Plant | Plant Operator | 12.0 | Regular Shift |

**Roster Analysis**:
- Total Rostered Staff: 10 personnel
- Day Shift: 6 scheduled (5 active, 1 on leave)
- Night Shift: 4 scheduled
- Total Planned Work Hours: 102.5 hours
- Pit 3 Coverage: 5 operators across 2 shifts
- Planned Absences: 12 hours (Tom Lee on leave)

---

## Related Documentation
- **ERD_07_Audit_Activity.md** - ERD diagram showing roster and activity relationship context
- **Dim_Shifts.md** - Shift dimension for roster context
- **Dim_Users_AssignedTo.md** - User dimension for personnel
- **Dim_Teams.md** - Team dimension (may align with roster groups)

---

## Change History
| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-15 | Auto-generated | Initial documentation from TMDL metadata |

---

## Notes
- **Planned Rosters**: This table stores *planned* rosters (scheduled shifts), not actual attendance or time tracking. Actual attendance would be in a separate fact table.
- **Soft Delete Filtering**: The WHERE clause filters to `IsDeleted = 0`, implementing soft deletes. Cancelled or superseded rosters have IsDeleted = 1 and are excluded.
- **12-Hour Default**: The Planned_Shift_Hour calculated column defaults to 12 hours when start/end datetimes are missing, suggesting 12-hour shifts are standard.
- **SAP Integration**: Multiple SAP-specific fields (Personnel_No, SAP_WSR_Description, SAP_Work_SCHEDULERule, Personnel_Area) indicate integration with SAP ERP for HR/payroll.
- **Name Duplication**: User names (First_Name, Surname, Second_Name) are duplicated from the user dimension, likely for performance (avoiding joins) or source system denormalization.
- **Planned Attendance/Absence**: Both fields are doubles, likely representing hours (not percentages). The sum often equals shift duration.
- **Operating_Area Classification**: Enables location-based roster management and capacity planning for multi-site operations.
- **Position and Process Group**: Organizational classification fields for skill-based analysis and process coverage reporting.
- **Point_Code Usage**: The meaning of Point_Code is not explicitly defined but may link to assignment points, time clocks, or geographic locations.
- **Attendance Classification**: Three attendance fields (Name, Type, Reference) provide flexible attendance categorization for payroll and reporting.
- **No Incremental Refresh**: Standard import mode suggests roster data is refreshed in full or historical rosters are managed at source.
- **Shift_Id Relationship**: Links to Dim_Shifts for standard shift definitions, enabling comparison between planned and standard shift durations.
- **DATEDIFF Precision**: Planned_Shift_Hour calculation uses MINUTE granularity then divides by 60, providing decimal hour precision (e.g., 10.5 hours).
- **Shift_Hour vs Planned_Shift_Hour**: Comparing these columns identifies roster entries with non-standard shift durations (overtime, partial shifts, extended shifts).
- **Personnel_No Uniqueness**: Personnel_No from SAP may serve as an alternate key to User_Id for cross-system reconciliation.
- **Tenant Isolation**: Standard tenant filtering applied in Power Query.
- **Created_Date Tracking**: Enables tracking when roster entries were created (may differ from Planned_Start_Datetime).
- **Last_Updated Tracking**: Captures roster modifications (schedule changes, cancellations, updates).
