# Assignment_Details_Snapshot

## Table Overview
`Assignment_Details_Snapshot` is an incremental refresh fact table that captures daily snapshots of assignment state changes, implementing a Slowly Changing Dimension (SCD) Type 2 pattern for historical tracking. Each row represents a snapshot of an assignment's status, assigned user, assigned team, and date range at a specific point in time.

This table enables historical analysis of assignment state changes, workload distribution over time, and team/user assignment patterns.

**Current Status**: Incremental Refresh enabled with 3-year rolling window and 2-day increments based on `Snapshot_Timestamp`.

---

## Specifications
- **Source**: Data warehouse snapshot table
- **Row Count**: High volume (daily snapshots × assignments × 3-year window)
- **Grain**: One row per assignment state snapshot
- **Primary Key**: Composite (Assignment_Id + Snapshot_Timestamp)
- **Incremental Refresh**: 3 years rolling, 2-day increments, filtered on `Snapshot_Timestamp`
- **Partitioning Strategy**: Incremental refresh by snapshot timestamp
- **Source Columns**: 8
- **Calculated Columns**: 0

---

## Column Specifications

| Column Name | Data Type | Format | Nullable | Hidden | Description |
|------------|-----------|--------|----------|--------|-------------|
| Assignment_Id | Int64 | | No | No | Foreign key to assignment being tracked |
| Status_Id | Int64 | | No | No | Foreign key to assignment status at snapshot time |
| Snapshot_Timestamp | DateTime | yyyy-MM-dd HH:mm:ss | No | No | Timestamp when this snapshot was captured (incremental refresh key) |
| User_Id | Int64 | | Yes | No | Foreign key to user assigned at snapshot time |
| Team_Id | Int64 | | Yes | No | Foreign key to team assigned at snapshot time |
| Assigned_To | String | | Yes | No | Text representation of assignment target (user or team name) |
| From_Date | DateTime | yyyy-MM-dd HH:mm:ss | Yes | No | Start date of the assignment period at snapshot time |
| To_Date | DateTime | yyyy-MM-dd HH:mm:ss | Yes | No | End date of the assignment period at snapshot time |

---

## Calculated Columns
None. This table uses only source columns from the data warehouse snapshot process.

---

## Relationships

### Outbound Relationships
| To Table | From Column(s) | To Column(s) | Cardinality | Cross Filter | Relationship ID |
|----------|---------------|--------------|-------------|--------------|-----------------|
| `Assignments` | Assignment_Id | Assignment_Id | Many-to-One | Single | (Assignment context) |
| `Dim_Assignment_Status` | Status_Id | Status_Id | Many-to-One | Single | (Status at snapshot time) |
| `Dim_Users_AssignedTo` | User_Id | User_Id | Many-to-One | Single | (Assigned user context - role-playing) |
| `Dim_Teams` | Team_Id | Team_Id | Many-to-One | Single | (Assigned team context) |

### Inbound Relationships
None. This is a leaf table in the relationship hierarchy.

---

## Power Query M Source

```m
let
    Source = Value.NativeQuery(
        Obzervr_DataWarehouse,
        "SELECT 
            Assignment_Id,
            Status_Id,
            Snapshot_Timestamp,
            User_Id,
            Team_Id,
            Assigned_To,
            From_Date,
            To_Date,
            TenantId
        FROM [dbo].[Assignment_Details_Snapshot]"
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
            {"Assignment_Id", Int64.Type},
            {"Status_Id", Int64.Type},
            {"Snapshot_Timestamp", type datetime},
            {"User_Id", Int64.Type},
            {"Team_Id", Int64.Type},
            {"Assigned_To", type text},
            {"From_Date", type datetime},
            {"To_Date", type datetime}
        }
    ),
    #"Filtered by Incremental Refresh" = Table.SelectRows(
        #"Changed Types",
        each [Snapshot_Timestamp] >= RangeStart and [Snapshot_Timestamp] < RangeEnd
    )
in
    #"Filtered by Incremental Refresh"
```

---

## DAX Query Patterns

### Example 1: Assignment State Changes Over Time
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_Date_Reference[Calendar_Year],
    Dim_Date_Reference[Calendar_Month_Name],
    Dim_Assignment_Status[Status_Name],
    "Snapshot_Count", COUNTROWS(Assignment_Details_Snapshot),
    "Unique_Assignments", DISTINCTCOUNT(Assignment_Details_Snapshot[Assignment_Id])
)
ORDER BY Dim_Date_Reference[Calendar_Year], Dim_Date_Reference[Month_Number], Dim_Assignment_Status[Status_Name]
```

### Example 2: User Workload Distribution at Point in Time
```dax
EVALUATE
VAR SnapshotDate = DATE(2024, 12, 1)
RETURN
    SUMMARIZECOLUMNS(
        Dim_Users_AssignedTo[Full_Name],
        FILTER(
            ALL(Assignment_Details_Snapshot),
            Assignment_Details_Snapshot[Snapshot_Timestamp] = SnapshotDate
        ),
        "Active_Assignments", COUNTROWS(Assignment_Details_Snapshot),
        "Status_Distribution", CONCATENATEX(
            VALUES(Dim_Assignment_Status[Status_Name]),
            Dim_Assignment_Status[Status_Name],
            ", "
        )
    )
ORDER BY [Active_Assignments] DESC
```

### Example 3: Assignment Duration Analysis from Snapshots
```dax
EVALUATE
TOPN(
    100,
    SELECTCOLUMNS(
        Assignment_Details_Snapshot,
        "Assignment_Id", Assignment_Details_Snapshot[Assignment_Id],
        "Assignment_Name", RELATED(Assignments[Assignment_Name]),
        "Status", RELATED(Dim_Assignment_Status[Status_Name]),
        "Assigned_To", Assignment_Details_Snapshot[Assigned_To],
        "From_Date", Assignment_Details_Snapshot[From_Date],
        "To_Date", Assignment_Details_Snapshot[To_Date],
        "Duration_Days", DATEDIFF(
            Assignment_Details_Snapshot[From_Date],
            Assignment_Details_Snapshot[To_Date],
            DAY
        ),
        "Snapshot_Time", Assignment_Details_Snapshot[Snapshot_Timestamp]
    ),
    Assignment_Details_Snapshot[Snapshot_Timestamp],
    DESC
)
```

### Example 4: Team Assignment History
```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Assignment_Details_Snapshot,
        "Team", RELATED(Dim_Teams[Name]),
        "Assignment", RELATED(Assignments[Assignment_Name]),
        "Status", RELATED(Dim_Assignment_Status[Status_Name]),
        "From_Date", Assignment_Details_Snapshot[From_Date],
        "To_Date", Assignment_Details_Snapshot[To_Date],
        "Snapshot_Date", FORMAT(Assignment_Details_Snapshot[Snapshot_Timestamp], "yyyy-MM-dd")
    ),
    NOT ISBLANK(Assignment_Details_Snapshot[Team_Id])
)
ORDER BY Assignment_Details_Snapshot[Snapshot_Timestamp] DESC
```

### Example 5: Status Transition Detection
```dax
EVALUATE
VAR SnapshotsWithPrevious = 
    ADDCOLUMNS(
        Assignment_Details_Snapshot,
        "Previous_Status_Id", 
            CALCULATE(
                MAX(Assignment_Details_Snapshot[Status_Id]),
                FILTER(
                    ALL(Assignment_Details_Snapshot),
                    Assignment_Details_Snapshot[Assignment_Id] = EARLIER(Assignment_Details_Snapshot[Assignment_Id])
                    && Assignment_Details_Snapshot[Snapshot_Timestamp] < EARLIER(Assignment_Details_Snapshot[Snapshot_Timestamp])
                )
            )
    )
RETURN
    FILTER(
        SELECTCOLUMNS(
            SnapshotsWithPrevious,
            "Assignment", RELATED(Assignments[Assignment_Name]),
            "From_Status", LOOKUPVALUE(
                Dim_Assignment_Status[Status_Name],
                Dim_Assignment_Status[Status_Id],
                [Previous_Status_Id]
            ),
            "To_Status", RELATED(Dim_Assignment_Status[Status_Name]),
            "Transition_Date", Assignment_Details_Snapshot[Snapshot_Timestamp],
            "Assigned_To", Assignment_Details_Snapshot[Assigned_To]
        ),
        NOT ISBLANK([Previous_Status_Id])
        && [Previous_Status_Id] <> Assignment_Details_Snapshot[Status_Id]
    )
ORDER BY Assignment_Details_Snapshot[Snapshot_Timestamp] DESC
```

### Example 6: Assignment Backlog Snapshot Trend
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_Date_Reference[Calendar_Week_Year],
    FILTER(
        ALL(Dim_Assignment_Status),
        Dim_Assignment_Status[Status_Name] IN {"Open", "In Progress"}
    ),
    "Backlog_Count", DISTINCTCOUNT(Assignment_Details_Snapshot[Assignment_Id]),
    "Unique_Users", DISTINCTCOUNT(Assignment_Details_Snapshot[User_Id]),
    "Unique_Teams", DISTINCTCOUNT(Assignment_Details_Snapshot[Team_Id])
)
ORDER BY Dim_Date_Reference[Calendar_Week_Year]
```

---

## Data Model Pattern

### SCD Type 2 Snapshot Pattern
`Assignment_Details_Snapshot` implements a Slowly Changing Dimension (SCD) Type 2 pattern through daily snapshot captures. Rather than tracking individual change events, this approach captures the complete state of assignments at regular intervals.

**Snapshot Characteristics**:
- **Daily Frequency**: Snapshots typically captured once per day
- **Complete State**: Each snapshot row contains the full assignment state (status, user, team, dates) at that point in time
- **Historical Reconstruction**: By filtering snapshots to a specific date, the exact assignment workload and state distribution at that moment can be reconstructed

**SCD Type 2 Implementation**:
Traditional SCD Type 2 tracks changes with effective-from and effective-to dates. This table uses `Snapshot_Timestamp` as the observation point, with `From_Date` and `To_Date` representing the assignment's scheduled period *as it was known at snapshot time*.

**Use Cases**:
1. **Historical Workload Analysis**: "How many assignments did User X have on December 1st, 2023?"
2. **Status Trend Analysis**: "Track the count of 'In Progress' assignments over the last quarter"
3. **Assignment Reassignment Tracking**: Detect when assignments moved between users or teams
4. **Backlog Trending**: Monitor how assignment backlogs grow or shrink over time
5. **Team Capacity Analysis**: Historical view of team workload distribution

**Snapshot vs. Current State**:
- The `Assignments` table contains current assignment state
- `Assignment_Details_Snapshot` contains historical states captured at specific points in time
- Changes not captured between snapshot intervals are not visible in this table

**Example Scenario - Assignment Lifecycle Tracking**:

Assignment ID 5001 - "Replace Conveyor Belt C-12"

| Snapshot_Timestamp | Status_Id | Assigned_To | User_Id | Team_Id | From_Date | To_Date |
|-------------------|-----------|-------------|---------|---------|-----------|---------|
| 2024-11-01 00:00 | 1 (Open) | Maintenance Team | NULL | 15 | 2024-11-05 | 2024-11-08 |
| 2024-11-02 00:00 | 1 (Open) | Maintenance Team | NULL | 15 | 2024-11-05 | 2024-11-08 |
| 2024-11-03 00:00 | 2 (Assigned) | John Smith | 42 | 15 | 2024-11-05 | 2024-11-08 |
| 2024-11-05 00:00 | 3 (In Progress) | John Smith | 42 | 15 | 2024-11-05 | 2024-11-08 |
| 2024-11-06 00:00 | 3 (In Progress) | John Smith | 42 | 15 | 2024-11-05 | 2024-11-10 |
| 2024-11-07 00:00 | 3 (In Progress) | John Smith | 42 | 15 | 2024-11-05 | 2024-11-10 |
| 2024-11-10 00:00 | 4 (Completed) | John Smith | 42 | 15 | 2024-11-05 | 2024-11-10 |

This sequence shows:
- Assignment created Nov 1st, unassigned to team
- Assigned to John Smith on Nov 3rd
- Started Nov 5th
- Due date extended from Nov 8 to Nov 10 (visible in Nov 6 snapshot)
- Completed Nov 10th

**Relationship to Current State**:
The most recent snapshot for each assignment should closely align with the current state in the `Assignments` table, though there may be a lag of up to 24 hours (or snapshot interval) before changes are captured.

---

## Related Documentation
- **ERD_04_Snapshots_Details.md** - ERD diagram showing relationship context
- **Assignments.md** - Current assignment state table
- **Dim_Assignment_Status.md** - Status dimension for state classification
- **Dim_Users_AssignedTo.md** - User dimension (role-playing for assigned users)
- **Dim_Teams.md** - Team dimension for team assignment context

---

## Change History
| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-15 | Auto-generated | Initial documentation from TMDL metadata |

---

## Notes
- **SCD Type 2 Pattern**: This table implements Slowly Changing Dimension Type 2 tracking through periodic snapshots rather than event-driven change capture. Each snapshot represents the complete assignment state at a specific point in time.
- **2-Day Incremental Periods**: The 2-day increment period means partitions are created every 2 days, balancing refresh performance with partition granularity.
- **3-Year Rolling Window**: Older snapshots beyond 3 years are automatically archived/removed, providing a reasonable historical analysis window while controlling model size.
- **Composite Key**: Primary key is the combination of `Assignment_Id` and `Snapshot_Timestamp`, allowing multiple snapshots per assignment over time.
- **User vs Team Assignment**: Either `User_Id` or `Team_Id` may be populated (or both). The `Assigned_To` text field provides a friendly display name regardless of assignment type.
- **From_Date and To_Date**: These represent the assignment's scheduled period *as it was at snapshot time*. If an assignment's due date is extended, subsequent snapshots will show the new `To_Date`.
- **Snapshot Frequency**: While typically daily, the exact snapshot frequency is controlled by the source system's snapshot process, not this table definition.
- **NULL User_Id**: When assignments are assigned to teams rather than individual users, `User_Id` will be NULL and `Team_Id` will be populated.
- **Status Transitions**: To detect status changes, compare `Status_Id` values across consecutive snapshots for the same `Assignment_Id`.
- **Historical Workload Queries**: To analyze workload at a specific date, filter snapshots to the nearest `Snapshot_Timestamp` on or before that date.
- **TenantId Removal**: The source includes `TenantId` for filtering, but this column is removed after tenant filtering in Power Query, as tenant context is maintained at the model level.
- **Performance Considerations**: The 2-day increment period creates more partitions than monthly or yearly increments, but provides faster refresh performance by limiting the amount of data processed per refresh cycle.
