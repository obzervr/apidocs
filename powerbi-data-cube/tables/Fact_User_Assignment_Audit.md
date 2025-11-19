# Fact_User_Assignment_Audit

## Table Overview
`Fact_User_Assignment_Audit` is an incremental refresh fact table that tracks user commands executed on assignments, providing an audit trail of assignment actions. Each row represents a single command fired by a user on an assignment, enabling action tracking, performance analysis, and compliance auditing.

This table supports user activity analysis beyond simple logons, capturing specific assignment-related commands.

**Current Status**: Incremental Refresh enabled based on `Fired_Timestamp`. Table load conditional on `EnableAssignmentUserAudit` parameter.

---

## Specifications
- **Source**: `FactAuditsUserAssignment` table
- **Row Count**: High volume (one row per user command on assignments)
- **Grain**: One row per user command per assignment
- **Primary Key**: Not explicitly defined (likely composite: User_Id + Assignment_Id + Fired_Timestamp)
- **Incremental Refresh**: Enabled on `Fired_Timestamp` (duration and increment not specified in TMDL)
- **Partitioning Strategy**: Incremental refresh by fired timestamp
- **Source Columns**: 5
- **Calculated Columns**: 0
- **Parameter Dependency**: `EnableAssignmentUserAudit` (table load conditional)

---

## Column Specifications

| Column Name | Data Type | Format | Nullable | Hidden | Description |
|------------|-----------|--------|----------|--------|-------------|
| User_Id | Int64 | | No | No | Foreign key to user dimension |
| Command_Name | String | | No | No | Name of the command executed (e.g., "CreateAssignment", "CompleteAssignment", "UpdateStatus") |
| Assignment_Id | Int64 | | No | No | Foreign key to assignment dimension |
| Fired_Timestamp | DateTime | yyyy-MM-dd HH:mm:ss | No | No | Timestamp when command was executed (incremental refresh key) |
| LastLoaded | DateTime | yyyy-MM-dd HH:mm:ss | Yes | No | ETL timestamp for data lineage |

---

## Calculated Columns
None. This table uses only source columns from the fact table.

---

## Relationships

### Outbound Relationships
| To Table | From Column(s) | To Column(s) | Cardinality | Cross Filter | Relationship ID |
|----------|---------------|--------------|-------------|--------------|-----------------|
| `Dim_User_Reference` | User_Id | User_Id | Many-to-One | Single | (User context) |
| `Assignments` | Assignment_Id | Assignment_Id | Many-to-One | Single | (Assignment context) |

### Inbound Relationships
None. This is a leaf table in the relationship hierarchy.

---

## Power Query M Source

```m
let
    Source = if EnableAssignmentUserAudit = true then
        Value.NativeQuery(
            Obzervr_DataWarehouse,
            "SELECT 
                User_Id,
                Command_Name,
                Assignment_Id,
                Fired_Timestamp,
                LastLoaded
            FROM [dbo].[FactAuditsUserAssignment]"
        )
    else
        #table(
            {"User_Id", "Command_Name", "Assignment_Id", "Fired_Timestamp", "LastLoaded"},
            {}
        ),
    
    #"Removed TenantId and Id Columns" = Table.RemoveColumns(
        Source,
        {"TenantId", "Id"}
    ),
    #"Filtered Tenant" = Table.SelectRows(
        #"Removed TenantId and Id Columns",
        each [TenantId] = TenantId  // Note: This appears to be a logical inconsistency in the documented steps
    ),
    #"Changed Types" = Table.TransformColumnTypes(
        #"Filtered Tenant",
        {
            {"User_Id", Int64.Type},
            {"Command_Name", type text},
            {"Assignment_Id", Int64.Type},
            {"Fired_Timestamp", type datetime},
            {"LastLoaded", type datetime}
        }
    )
in
    #"Changed Types"

// Note: The Power Query steps appear to have a logical issue - TenantId is removed before filtering by it.
// Actual implementation likely filters before removal.
```

---

## DAX Query Patterns

### Example 1: Command Frequency by Type
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Fact_User_Assignment_Audit[Command_Name],
    "Command_Count", COUNTROWS(Fact_User_Assignment_Audit),
    "Unique_Users", DISTINCTCOUNT(Fact_User_Assignment_Audit[User_Id]),
    "Unique_Assignments", DISTINCTCOUNT(Fact_User_Assignment_Audit[Assignment_Id]),
    "First_Command", MIN(Fact_User_Assignment_Audit[Fired_Timestamp]),
    "Latest_Command", MAX(Fact_User_Assignment_Audit[Fired_Timestamp])
)
ORDER BY [Command_Count] DESC
```

### Example 2: User Activity Report
```dax
EVALUATE
TOPN(
    50,
    ADDCOLUMNS(
        SUMMARIZE(
            Fact_User_Assignment_Audit,
            Fact_User_Assignment_Audit[User_Id]
        ),
        "User_Name", RELATED(Dim_User_Reference[Full_Name]),
        "Total_Commands", COUNTROWS(Fact_User_Assignment_Audit),
        "Unique_Command_Types", DISTINCTCOUNT(Fact_User_Assignment_Audit[Command_Name]),
        "Assignments_Touched", DISTINCTCOUNT(Fact_User_Assignment_Audit[Assignment_Id]),
        "First_Activity", MIN(Fact_User_Assignment_Audit[Fired_Timestamp]),
        "Latest_Activity", MAX(Fact_User_Assignment_Audit[Fired_Timestamp]),
        "Most_Common_Command", CALCULATE(
            TOPN(1, VALUES(Fact_User_Assignment_Audit[Command_Name]), COUNTROWS(Fact_User_Assignment_Audit), DESC),
            ALLEXCEPT(Fact_User_Assignment_Audit, Fact_User_Assignment_Audit[User_Id])
        )
    ),
    [Total_Commands],
    DESC
)
```

### Example 3: Assignment Action Timeline
```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Fact_User_Assignment_Audit,
        "Assignment_Name", RELATED(Assignments[Assignment_Name]),
        "User_Name", RELATED(Dim_User_Reference[Full_Name]),
        "Command", Fact_User_Assignment_Audit[Command_Name],
        "Executed_At", FORMAT(Fact_User_Assignment_Audit[Fired_Timestamp], "yyyy-MM-dd HH:mm:ss"),
        "Assignment_Status", RELATED(Dim_Assignment_Status[Status_Name])
    ),
    Fact_User_Assignment_Audit[Assignment_Id] = 12345
)
ORDER BY Fact_User_Assignment_Audit[Fired_Timestamp]
```

### Example 4: Command Execution Trend
```dax
EVALUATE
SUMMARIZECOLUMNS(
    ROLLUPADDISSUBTOTAL(
        Fact_User_Assignment_Audit[Command_Name],
        "Is_Summary", "Command_Summary"
    ),
    Dim_Date_Reference[Calendar_Year],
    Dim_Date_Reference[Calendar_Month_Name],
    "Command_Count", COUNTROWS(Fact_User_Assignment_Audit),
    "Unique_Users", DISTINCTCOUNT(Fact_User_Assignment_Audit[User_Id])
)
ORDER BY Dim_Date_Reference[Calendar_Year], Dim_Date_Reference[Month_Number], Fact_User_Assignment_Audit[Command_Name]
```

### Example 5: Peak Activity Hours
```dax
EVALUATE
ADDCOLUMNS(
    SUMMARIZE(
        Fact_User_Assignment_Audit,
        "Hour", HOUR(Fact_User_Assignment_Audit[Fired_Timestamp])
    ),
    "Command_Count", COUNTROWS(Fact_User_Assignment_Audit),
    "Unique_Users", DISTINCTCOUNT(Fact_User_Assignment_Audit[User_Id]),
    "Top_Commands", CONCATENATEX(
        TOPN(3, VALUES(Fact_User_Assignment_Audit[Command_Name]), COUNTROWS(Fact_User_Assignment_Audit), DESC),
        Fact_User_Assignment_Audit[Command_Name],
        ", "
    )
)
ORDER BY [Hour]
```

### Example 6: User Command Performance
```dax
EVALUATE
ADDCOLUMNS(
    SUMMARIZE(
        Fact_User_Assignment_Audit,
        Fact_User_Assignment_Audit[User_Id],
        Fact_User_Assignment_Audit[Command_Name]
    ),
    "User_Name", RELATED(Dim_User_Reference[Full_Name]),
    "Command_Count", COUNTROWS(Fact_User_Assignment_Audit),
    "First_Execution", MIN(Fact_User_Assignment_Audit[Fired_Timestamp]),
    "Latest_Execution", MAX(Fact_User_Assignment_Audit[Fired_Timestamp]),
    "Assignments_Affected", DISTINCTCOUNT(Fact_User_Assignment_Audit[Assignment_Id])
)
ORDER BY Fact_User_Assignment_Audit[User_Id], [Command_Count] DESC
```

---

## Data Model Pattern

### Command Audit Trail Pattern
`Fact_User_Assignment_Audit` implements a command audit trail pattern that captures discrete user actions on assignments. This pattern provides a detailed activity log beyond simple logon tracking, recording specific operations performed by users.

**Audit Trail Characteristics**:
- **Command-Based**: Captures specific commands/actions, not just authentication events
- **Assignment Context**: Links commands to specific assignments being acted upon
- **User Attribution**: Tracks which user executed each command
- **Timestamp Precision**: Full datetime for temporal analysis and reconstruction
- **Lightweight**: Only 5 columns, focused on essential audit data

**Command_Name Examples** (inferred based on typical command patterns):
- **CreateAssignment**: User created a new assignment
- **UpdateAssignment**: User modified assignment details
- **CompleteAssignment**: User marked assignment as complete
- **StartAssignment**: User began work on assignment
- **RescheduleAssignment**: User changed assignment dates
- **AddComment**: User added comment to assignment
- **AttachFile**: User uploaded attachment
- **UpdateStatus**: User changed assignment status
- **AssignUser**: User assigned/reassigned assignment to someone
- **DeleteAssignment**: User deleted assignment (soft delete)

The actual command set depends on the Obzervr application's command structure.

**Audit Trail Use Cases**:
1. **Compliance**: Track who made what changes and when
2. **Performance Analysis**: Measure user productivity by command counts
3. **Workflow Analysis**: Understand assignment lifecycle through command sequences
4. **User Training**: Identify users who may need additional training based on command patterns
5. **System Usage**: Track feature utilization by command type
6. **Troubleshooting**: Reconstruct sequence of events leading to issues

**Incremental Refresh Pattern**:
The table uses incremental refresh on `Fired_Timestamp`:
- Keeps recent command history accessible
- Archives older audit data
- Balances compliance needs with model performance
- Duration and increment not specified in TMDL (likely 1-5 year rolling window)

**Parameter-Based Loading**:
The `EnableAssignmentUserAudit` parameter controls loading:
- **TRUE**: Full audit trail loaded
- **FALSE**: Empty table with schema only

This pattern enables:
- Deployment-specific configuration (disable for basic deployments)
- Performance optimization (skip if auditing not required)
- Compliance configuration (enable only when needed)
- Licensing/feature gating

**Relationship to Logon Activities**:
This table complements `Fact_User_LogOn_Activities`:
- **Logon Activities**: Authentication events (when users log in)
- **Assignment Audit**: Action events (what users do after logging in)

Together they provide complete activity tracking.

**Removed Columns**:
The Power Query removes `TenantId` and `Id` columns from source:
- **TenantId**: Removed after filtering (standard pattern)
- **Id**: Removed, suggesting composite primary key is sufficient (User_Id + Assignment_Id + Fired_Timestamp)

**No Date_Key**:
Unlike many fact tables, this table doesn't include a calculated Date_Key. Date analysis uses the `Fired_Timestamp` directly or requires relationship through assignment's date context.

**Example Scenario - Assignment Lifecycle Audit Trail**:

**Assignment 5001** - "Replace Conveyor Belt C-12" command audit trail:

| Fired_Timestamp | User_Name | Command_Name | Context |
|----------------|-----------|--------------|---------|
| 2024-11-01 08:15:23 | Sarah Manager | CreateAssignment | Assignment created |
| 2024-11-01 08:17:45 | Sarah Manager | AssignUser | Assigned to Maintenance Team |
| 2024-11-03 06:52:10 | John Smith | AcceptAssignment | Technician accepted assignment |
| 2024-11-03 06:53:22 | John Smith | AddComment | "Inspecting conveyor belt" |
| 2024-11-03 07:10:05 | John Smith | StartAssignment | Work started |
| 2024-11-03 09:22:18 | John Smith | AttachFile | Photo of damage uploaded |
| 2024-11-03 09:25:33 | John Smith | AddComment | "Requires new belt and motor" |
| 2024-11-03 14:30:12 | Sarah Manager | UpdateAssignment | Priority increased |
| 2024-11-05 10:45:50 | John Smith | UpdateStatus | Status → In Progress |
| 2024-11-08 15:20:15 | John Smith | AttachFile | Post-repair photos |
| 2024-11-08 15:25:40 | John Smith | CompleteAssignment | Work completed |
| 2024-11-08 16:10:22 | Sarah Manager | ApproveAssignment | Manager approval |

**Analysis from Audit Trail**:
- **Creation to Acceptance**: 2 days, 22 hours (delay in assignment pickup)
- **Work Duration**: 5 days, 8 hours (Start to Complete)
- **Total Lifecycle**: 7 days, 7 hours (Create to Approve)
- **User Interactions**: 2 users (Manager + Technician)
- **Command Count**: 12 commands
- **Documentation**: 2 comments, 2 file attachments
- **Status Changes**: 2 explicit status updates

**User Activity Summary** - John Smith (November 2024):
- Total Commands: 487
- Unique Assignments: 45
- Most Common Commands:
  1. StartAssignment (45 times)
  2. CompleteAssignment (42 times)
  3. AddComment (112 times)
  4. AttachFile (78 times)
  5. UpdateStatus (35 times)
- Average Commands per Assignment: 10.8
- Peak Activity Hours: 07:00-08:00, 14:00-15:00

---

## Related Documentation
- **ERD_07_Audit_Activity.md** - ERD diagram showing audit and activity relationship context
- **Assignments.md** - Assignment table being audited
- **Dim_User_Reference.md** - User dimension for command attribution
- **Fact_User_LogOn_Activities.md** - Complementary authentication event tracking

---

## Change History
| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-15 | Auto-generated | Initial documentation from TMDL metadata |

---

## Notes
- **Command Audit Trail**: This table provides detailed audit trail of user commands executed on assignments, going beyond simple logon tracking to capture specific actions.
- **Parameter-Based Loading**: The table only loads when `EnableAssignmentUserAudit = true`, creating empty schema when false for deployment flexibility.
- **Incremental Refresh**: Uses incremental refresh on Fired_Timestamp, though duration and increment not specified in TMDL metadata.
- **Removed Columns**: Source columns TenantId and Id are removed in Power Query (TenantId after filtering, Id as redundant with composite key).
- **No Date_Key**: Unlike many fact tables, this doesn't include a calculated Date_Key column. Date analysis uses Fired_Timestamp directly.
- **Command_Name Values**: The actual set of command names depends on the Obzervr application's command structure and may evolve with application features.
- **Composite Primary Key**: While no explicit PK defined, the combination of User_Id + Assignment_Id + Fired_Timestamp likely serves as natural key (though users could theoretically fire multiple commands on same assignment at same second).
- **LastLoaded Timestamp**: ETL lineage tracking, enables refresh monitoring and data currency validation.
- **Complementary to Logons**: Works alongside Fact_User_LogOn_Activities to provide complete activity picture (authentication + actions).
- **Compliance Use Case**: Primary use case is compliance auditing, tracking who did what and when on assignments.
- **Performance Analysis**: Secondary use case is user performance and productivity analysis by command frequency.
- **Workflow Reconstruction**: Command sequences enable reconstruction of assignment lifecycle and user workflow patterns.
- **No Command Parameters**: The table captures command name but not command parameters or details (e.g., what specifically was updated in UpdateAssignment).
- **User_Id vs Dim_User_Reference**: Relationship to Dim_User_Reference (role-playing dimension) rather than Dim_Users_AssignedTo, appropriate for audit context.
- **No Outcome Tracking**: The table doesn't capture whether commands succeeded or failed - only that they were fired.
- **Tenant Filtering**: Standard tenant filtering pattern applied (though Power Query steps shown appear to have logical inconsistency - likely filters before removing TenantId).
- **Timezone Considerations**: Fired_Timestamp timezone handling important for multi-region deployments with distributed users.
- **Command Evolution**: As Obzervr application evolves and adds features, new command types will appear in Command_Name without schema changes.
