# Dim_Teams

## Table Overview
`Dim_Teams` is a dimension table that defines teams within the Obzervr system. Each row represents a single team, which can have multiple users assigned through the `Team_Users` bridge table. Teams provide organizational grouping for assignments, workload management, and reporting.

This table includes conditional column removal based on the `UseTeamResourceCentre` parameter, allowing deployment-specific configuration.

**Current Status**: Standard dimension table with parameter-based column filtering.

---

## Specifications
- **Source**: `VW_Teams` view
- **Row Count**: Low volume (typically 10-100 teams per tenant)
- **Grain**: One row per team
- **Primary Key**: `Team_Id`
- **Incremental Refresh**: Not enabled
- **Partitioning Strategy**: Standard import
- **Source Columns**: 6 (+ 2 conditional columns removed based on parameter)
- **Calculated Columns**: 0
- **Parameter Dependency**: `UseTeamResourceCentre` controls column inclusion

---

## Column Specifications

| Column Name | Data Type | Format | Nullable | Hidden | Description |
|------------|-----------|--------|----------|--------|-------------|
| Team_Id | Int64 | | No | No | Primary key for team identification |
| Name | String | | No | No | Team display name |
| Description | String | | Yes | No | Team purpose or role description |
| Created_Date | DateTime | yyyy-MM-dd HH:mm:ss | No | No | Timestamp when team was created |
| Last_Updated | DateTime | yyyy-MM-dd HH:mm:ss | Yes | No | Timestamp of last modification |
| Tenant_Id | Int64 | | No | No | Foreign key to tenant dimension |

**Conditionally Removed Columns** (removed when `UseTeamResourceCentre` = false):
- `Work_Centre`: String - Work center classification for team
- `Resource_Centre`: String - Resource center classification for team

---

## Calculated Columns
None. This table uses only source columns from the view.

---

## Relationships

### Outbound Relationships
| To Table | From Column(s) | To Column(s) | Cardinality | Cross Filter | Relationship ID |
|----------|---------------|--------------|-------------|--------------|-----------------|
| `Dim_Tenant` | Tenant_Id | Tenant_Id | Many-to-One | Single | (Tenant context) |

### Inbound Relationships
| From Table | From Column(s) | To Column(s) | Cardinality | Cross Filter | Relationship ID |
|-----------|---------------|--------------|-------------|--------------|-----------------|
| `Team_Users` | Team_Id | Team_Id | Many-to-One | Single | (Bridge to users) |
| `Assignment_Details_Snapshot` | Team_Id | Team_Id | Many-to-One | Single | (Assignment history) |
| `Fact_Rosters` | Shift_Id | Team_Id | Many-to-One | Single | (Note: May be Team_Id in source despite column name) |

---

## Power Query M Source

```m
let
    Source = Value.NativeQuery(
        Obzervr_DataWarehouse,
        "SELECT 
            Team_Id,
            Name,
            Description,
            Created_Date,
            Last_Updated,
            Tenant_Id" &
            (if UseTeamResourceCentre then ",
            Work_Centre,
            Resource_Centre" else "") &
        " FROM [dbo].[VW_Teams]"
    ),
    #"Filtered Tenant" = Table.SelectRows(
        Source,
        each [Tenant_Id] = TenantId
    ),
    #"Changed Types" = Table.TransformColumnTypes(
        #"Filtered Tenant",
        {
            {"Team_Id", Int64.Type},
            {"Name", type text},
            {"Description", type text},
            {"Created_Date", type datetime},
            {"Last_Updated", type datetime},
            {"Tenant_Id", Int64.Type}
        } & 
        (if UseTeamResourceCentre then {
            {"Work_Centre", type text},
            {"Resource_Centre", type text}
        } else {})
    )
in
    #"Changed Types"
```

---

## DAX Query Patterns

### Example 1: Team Listing with User Counts
```dax
EVALUATE
ADDCOLUMNS(
    Dim_Teams,
    "Member_Count", CALCULATE(
        COUNTROWS(Team_Users),
        Team_Users[Is_Member] = TRUE()
    ),
    "Supervisor_Count", CALCULATE(
        COUNTROWS(Team_Users),
        Team_Users[Is_Supervisor] = TRUE()
    ),
    "Total_Users", DISTINCTCOUNT(Team_Users[User_Id])
)
ORDER BY [Total_Users] DESC
```

### Example 2: Team Assignment Workload
```dax
EVALUATE
ADDCOLUMNS(
    Dim_Teams,
    "Current_Assignments", CALCULATE(
        DISTINCTCOUNT(Assignments[Assignment_Id]),
        FILTER(
            Assignments,
            RELATED(Dim_Assignment_Status[Status_Name]) IN {"Open", "In Progress"}
        )
    ),
    "Completed_Assignments", CALCULATE(
        DISTINCTCOUNT(Assignments[Assignment_Id]),
        FILTER(
            Assignments,
            RELATED(Dim_Assignment_Status[Status_Name]) = "Completed"
        )
    )
)
ORDER BY [Current_Assignments] DESC
```

### Example 3: Team Details Report
```dax
EVALUATE
SELECTCOLUMNS(
    Dim_Teams,
    "Team_Id", Dim_Teams[Team_Id],
    "Team_Name", Dim_Teams[Name],
    "Description", Dim_Teams[Description],
    "Created", FORMAT(Dim_Teams[Created_Date], "yyyy-MM-dd"),
    "Days_Active", DATEDIFF(Dim_Teams[Created_Date], TODAY(), DAY)
)
ORDER BY Dim_Teams[Name]
```

### Example 4: Team Membership Matrix
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_Teams[Name],
    Dim_Users_AssignedTo[Full_Name],
    TREATAS(
        FILTER(Team_Users, Team_Users[Is_Member] = TRUE()),
        Team_Users[Team_Id],
        Team_Users[User_Id]
    ),
    "Is_Supervisor", MAX(Team_Users[Is_Supervisor])
)
ORDER BY Dim_Teams[Name], Dim_Users_AssignedTo[Full_Name]
```

### Example 5: Team Activity Timeline
```dax
EVALUATE
ADDCOLUMNS(
    Dim_Teams,
    "Created_Date", Dim_Teams[Created_Date],
    "Last_Updated", Dim_Teams[Last_Updated],
    "Days_Since_Update", DATEDIFF(Dim_Teams[Last_Updated], TODAY(), DAY),
    "First_Assignment", CALCULATE(
        MIN(Assignments[Created_Date]),
        USERELATIONSHIP(Assignments[Team_Id], Dim_Teams[Team_Id])
    ),
    "Latest_Assignment", CALCULATE(
        MAX(Assignments[Created_Date]),
        USERELATIONSHIP(Assignments[Team_Id], Dim_Teams[Team_Id])
    )
)
ORDER BY Dim_Teams[Created_Date] DESC
```

### Example 6: Teams Without Members
```dax
EVALUATE
SELECTCOLUMNS(
    FILTER(
        Dim_Teams,
        CALCULATE(COUNTROWS(Team_Users)) = 0
    ),
    "Team_Id", Dim_Teams[Team_Id],
    "Team_Name", Dim_Teams[Name],
    "Description", Dim_Teams[Description],
    "Created_Date", Dim_Teams[Created_Date],
    "Days_Inactive", DATEDIFF(Dim_Teams[Created_Date], TODAY(), DAY)
)
ORDER BY Dim_Teams[Created_Date]
```

---

## Data Model Pattern

### Team Organization Pattern
`Dim_Teams` serves as the core dimension for team-based organization and assignment management. Teams act as containers for users and can be assigned work assignments, enabling workload distribution and team-based reporting.

**Team Structure**:
- **Team Definition**: `Team_Id` and `Name` uniquely identify each team
- **Description**: Provides context about team purpose, skills, or coverage area
- **Many-to-Many User Relationships**: Teams connect to users through the `Team_Users` bridge table
- **Assignment Ownership**: Teams can be assigned to work assignments (either directly or through team members)

**Team-User Relationship**:
The relationship between teams and users is many-to-many:
- A user can belong to multiple teams
- A team contains multiple users
- The `Team_Users` bridge table manages this relationship
- Users can have roles within teams (member, supervisor)

**Example Team Structure**:

| Team_Id | Name | Description | Member_Count | Supervisor_Count |
|---------|------|-------------|--------------|------------------|
| 15 | Maintenance Team A | Day shift mechanical maintenance | 8 | 2 |
| 16 | Maintenance Team B | Night shift mechanical maintenance | 6 | 1 |
| 17 | Electrical Team | 24/7 electrical maintenance | 10 | 3 |
| 18 | Operations - Pit 3 | Haul truck and excavator operators | 25 | 4 |

**Team Assignment Pattern**:
Teams can be used in multiple contexts:
1. **Direct Assignment**: Assignments assigned to a team rather than individual users
2. **Team Member Assignment**: Assignments distributed among team members
3. **Team Workload Reporting**: Aggregating assignments across all team members
4. **Shift Coverage**: Teams aligned with shift schedules for 24/7 operations

**Work Centre and Resource Centre** (Conditional):
When `UseTeamResourceCentre` parameter is enabled, two additional classification columns are available:
- **Work_Centre**: Physical location or operational area (e.g., "Workshop A", "Pit 3", "Processing Plant")
- **Resource_Centre**: Cost or resource allocation center for budget tracking

These columns enable:
- Location-based team filtering
- Cost center reporting
- Resource allocation analysis
- Work center capacity planning

The conditional removal of these columns when `UseTeamResourceCentre = false` suggests these are optional features that may not be used by all Obzervr deployments.

**Team Lifecycle**:
- `Created_Date`: When team was established
- `Last_Updated`: Most recent team configuration change (name, description, membership)
- Teams are typically long-lived (months/years) compared to assignments (days/weeks)

**Example Scenario - Shift-Based Maintenance Teams**:

Organization has 24/7 maintenance operations with three shift-based teams:

**Day Shift Maintenance Team** (7am - 3pm):
- Team_Id: 101
- Name: "Maintenance - Day Shift"
- Description: "Preventive maintenance and scheduled repairs"
- Members: 12 technicians, 2 supervisors
- Work_Centre: "Maintenance Workshop"
- Typical workload: Planned maintenance, inspections

**Swing Shift Maintenance Team** (3pm - 11pm):
- Team_Id: 102
- Name: "Maintenance - Swing Shift"
- Description: "Corrective maintenance and emergency response"
- Members: 8 technicians, 1 supervisor
- Work_Centre: "Maintenance Workshop"
- Typical workload: Reactive repairs, breakdown response

**Night Shift Maintenance Team** (11pm - 7am):
- Team_Id: 103
- Name: "Maintenance - Night Shift"
- Description: "Emergency breakdown coverage"
- Members: 4 technicians, 1 supervisor
- Work_Centre: "Maintenance Workshop"
- Typical workload: Emergency repairs, on-call response

Assignment distribution:
- High-priority breakdowns → Night Shift team (immediate response)
- Scheduled preventive maintenance → Day Shift team (planned work)
- Equipment failures during production → Swing Shift team (minimize downtime)

---

## Related Documentation
- **ERD_05_Users_Teams.md** - ERD diagram showing team and user relationship context
- **Team_Users.md** - Bridge table connecting teams to users
- **Assignment_Details_Snapshot.md** - Historical team assignment tracking
- **Dim_Users_AssignedTo.md** - Users who can be team members
- **Fact_Rosters.md** - Team shift scheduling

---

## Change History
| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-15 | Auto-generated | Initial documentation from TMDL metadata |

---

## Notes
- **Conditional Columns**: The `Work_Centre` and `Resource_Centre` columns are conditionally included based on the `UseTeamResourceCentre` parameter. When false, these columns are not queried or imported.
- **Parameter-Based Schema**: The Power Query M code dynamically builds the SQL query string and column type definitions based on the parameter value, demonstrating advanced Power Query pattern for deployment-specific configuration.
- **Team Definition Flexibility**: Teams can represent shift groups, skill specializations, geographic locations, or functional departments depending on organizational needs.
- **Many-to-Many Relationships**: Teams connect to users through the `Team_Users` bridge table, enabling users to belong to multiple teams simultaneously.
- **Assignment Context**: Teams can own assignments directly (team assignment) or indirectly (through team member assignments).
- **Supervisor Role**: The `Team_Users` table tracks supervisor status, enabling team hierarchy and approval workflows.
- **Tenant Isolation**: Teams are filtered by `Tenant_Id` in Power Query, maintaining multi-tenant data isolation.
- **Name Uniqueness**: While `Team_Id` is the primary key, team names are likely unique within each tenant for clarity.
- **Description Usage**: The description field provides context for team purpose but is nullable, suggesting some teams may not require detailed descriptions.
- **Team Lifecycle**: Teams typically have longer lifespans than assignments. The `Created_Date` and `Last_Updated` timestamps track team configuration history.
- **Empty Teams**: Teams without members (checked via `Team_Users` relationship) may represent planned teams, archived teams, or configuration placeholders.
- **Resource Centre Pattern**: When enabled, Work_Centre and Resource_Centre columns likely integrate with ERP or cost accounting systems for financial reporting.
- **No Soft Deletes**: This table doesn't have an IsDeleted flag, suggesting teams are either active (present) or permanently deleted (removed from view).
- **Assignment Relationship Note**: The relationship from `Fact_Rosters[Shift_Id]` may be a documentation error - verify if this should be `Roster[Team_Id]` or if Shift_Id serves dual purpose.
