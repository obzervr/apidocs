# Team_Users

## Table Overview
`Team_Users` is a bridge table that implements a many-to-many relationship between teams and users. Each row represents a user's membership in a team, including their role (member, supervisor) within that team. This table enables flexible team composition and supports users belonging to multiple teams simultaneously.

This bridge table pattern is essential for team-based workload distribution, reporting by team membership, and organizational hierarchy analysis.

**Current Status**: Standard bridge table (no incremental refresh).

---

## Specifications
- **Source**: `FactTeamUsers` table
- **Row Count**: Variable (depends on team composition complexity)
- **Grain**: One row per user-team membership
- **Primary Key**: Composite (Team_Id + User_Id) or (Team_Id + User_TenantId)
- **Incremental Refresh**: Not enabled
- **Partitioning Strategy**: Standard import
- **Source Columns**: 9
- **Calculated Columns**: 1 (Team name)

---

## Column Specifications

| Column Name | Data Type | Format | Nullable | Hidden | Description |
|------------|-----------|--------|----------|--------|-------------|
| Tenant_Id | Int64 | | No | No | Foreign key to tenant dimension |
| User_Id | Int64 | | No | No | Foreign key to user dimension |
| Team_Id | Int64 | | No | No | Foreign key to team dimension |
| Is_Supervisor | Boolean | true/false | No | No | Indicates if user has supervisor role within this team |
| Is_Member | Boolean | true/false | No | No | Indicates if user is an active member of this team |
| Created_Date | DateTime | yyyy-MM-dd HH:mm:ss | No | No | Timestamp when user was added to team |
| Last_Updated | DateTime | yyyy-MM-dd HH:mm:ss | Yes | No | Timestamp of last modification to membership |
| User_TenantId | String | | No | No | Composite key combining User_Id and Tenant_Id |
| LastLoaded | DateTime | yyyy-MM-dd HH:mm:ss | Yes | No | ETL timestamp for data lineage |

---

## Calculated Columns

### Team
Retrieves the team name from the `Dim_Teams` dimension for easier filtering and display in visuals.

```dax
Team = RELATED(Dim_Teams[Name])
```

---

## Relationships

### Outbound Relationships
| To Table | From Column(s) | To Column(s) | Cardinality | Cross Filter | Relationship ID |
|----------|---------------|--------------|-------------|--------------|-----------------|
| `Dim_Tenant` | Tenant_Id | Tenant_Id | Many-to-One | Single | (Tenant context) |
| `Dim_Users_AssignedTo` | User_Id | User_Id | Many-to-One | Single | (User context) |
| `Dim_Teams` | Team_Id | Team_Id | Many-to-One | Single | (Team context) |

### Inbound Relationships
None. This is a bridge table that connects dimensions but is not referenced by other tables.

---

## Power Query M Source

```m
let
    Source = Value.NativeQuery(
        Obzervr_DataWarehouse,
        "SELECT 
            Tenant_Id,
            User_Id,
            Team_Id,
            Is_Supervisor,
            Is_Member,
            Created_Date,
            Last_Updated,
            LastLoaded
        FROM [dbo].[FactTeamUsers]"
    ),
    #"Filtered Tenant" = Table.SelectRows(
        Source,
        each [Tenant_Id] = TenantId
    ),
    #"Changed Types" = Table.TransformColumnTypes(
        #"Filtered Tenant",
        {
            {"Tenant_Id", Int64.Type},
            {"User_Id", Int64.Type},
            {"Team_Id", Int64.Type},
            {"Is_Supervisor", type logical},
            {"Is_Member", type logical},
            {"Created_Date", type datetime},
            {"Last_Updated", type datetime},
            {"LastLoaded", type datetime}
        }
    ),
    #"Added User_TenantId" = Table.AddColumn(
        #"Changed Types",
        "User_TenantId",
        each Text.From([User_Id]) & "_" & Text.From([Tenant_Id]),
        type text
    )
in
    #"Added User_TenantId"
```

---

## DAX Query Patterns

### Example 1: User Team Memberships
```dax
EVALUATE
SELECTCOLUMNS(
    FILTER(
        Team_Users,
        Team_Users[Is_Member] = TRUE()
    ),
    "User_Name", RELATED(Dim_Users_AssignedTo[Full_Name]),
    "Team_Name", Team_Users[Team],
    "Is_Supervisor", Team_Users[Is_Supervisor],
    "Member_Since", FORMAT(Team_Users[Created_Date], "yyyy-MM-dd")
)
ORDER BY RELATED(Dim_Users_AssignedTo[Full_Name]), Team_Users[Team]
```

### Example 2: Team Composition Report
```dax
EVALUATE
ADDCOLUMNS(
    Dim_Teams,
    "Total_Members", CALCULATE(
        COUNTROWS(Team_Users),
        Team_Users[Is_Member] = TRUE()
    ),
    "Supervisor_Count", CALCULATE(
        COUNTROWS(Team_Users),
        Team_Users[Is_Supervisor] = TRUE()
    ),
    "Regular_Members", CALCULATE(
        COUNTROWS(Team_Users),
        Team_Users[Is_Member] = TRUE()
        && Team_Users[Is_Supervisor] = FALSE()
    ),
    "Member_Names", CALCULATE(
        CONCATENATEX(
            FILTER(Team_Users, Team_Users[Is_Member] = TRUE()),
            RELATED(Dim_Users_AssignedTo[Full_Name]),
            ", ",
            RELATED(Dim_Users_AssignedTo[Full_Name]), ASC
        )
    )
)
ORDER BY [Total_Members] DESC
```

### Example 3: Supervisors Across Multiple Teams
```dax
EVALUATE
FILTER(
    SUMMARIZECOLUMNS(
        Dim_Users_AssignedTo[Full_Name],
        FILTER(Team_Users, Team_Users[Is_Supervisor] = TRUE()),
        "Supervisor_In_Teams", COUNTROWS(Team_Users),
        "Team_Names", CONCATENATEX(
            VALUES(Team_Users[Team]),
            Team_Users[Team],
            ", "
        )
    ),
    [Supervisor_In_Teams] > 1
)
ORDER BY [Supervisor_In_Teams] DESC
```

### Example 4: Team Membership Timeline
```dax
EVALUATE
SELECTCOLUMNS(
    TOPN(
        100,
        Team_Users,
        Team_Users[Created_Date],
        DESC
    ),
    "User", RELATED(Dim_Users_AssignedTo[Full_Name]),
    "Team", Team_Users[Team],
    "Is_Supervisor", Team_Users[Is_Supervisor],
    "Is_Member", Team_Users[Is_Member],
    "Added_Date", FORMAT(Team_Users[Created_Date], "yyyy-MM-dd HH:mm"),
    "Last_Updated", FORMAT(Team_Users[Last_Updated], "yyyy-MM-dd HH:mm"),
    "Days_In_Team", DATEDIFF(Team_Users[Created_Date], TODAY(), DAY)
)
```

### Example 5: Users Not in Any Team
```dax
EVALUATE
SELECTCOLUMNS(
    FILTER(
        Dim_Users_AssignedTo,
        NOT CONTAINS(
            Team_Users,
            Team_Users[User_Id], Dim_Users_AssignedTo[User_Id]
        )
    ),
    "User_Id", Dim_Users_AssignedTo[User_Id],
    "Full_Name", Dim_Users_AssignedTo[Full_Name],
    "Email", Dim_Users_AssignedTo[Email],
    "Role", Dim_Users_AssignedTo[Role],
    "Department", Dim_Users_AssignedTo[Department]
)
ORDER BY Dim_Users_AssignedTo[Full_Name]
```

### Example 6: Team Member Status Analysis
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Team_Users[Team],
    "Active_Members", CALCULATE(
        COUNTROWS(Team_Users),
        Team_Users[Is_Member] = TRUE()
    ),
    "Inactive_Members", CALCULATE(
        COUNTROWS(Team_Users),
        Team_Users[Is_Member] = FALSE()
    ),
    "Supervisors", CALCULATE(
        COUNTROWS(Team_Users),
        Team_Users[Is_Supervisor] = TRUE()
        && Team_Users[Is_Member] = TRUE()
    ),
    "Latest_Addition", MAX(Team_Users[Created_Date]),
    "Last_Change", MAX(Team_Users[Last_Updated])
)
ORDER BY Team_Users[Team]
```

---

## Data Model Pattern

### Many-to-Many Bridge Table Pattern
`Team_Users` implements a classic bridge table pattern to enable many-to-many relationships between users and teams. This pattern is essential when entities on both sides of a relationship can have multiple associations.

**Many-to-Many Relationship**:
- **One User → Many Teams**: A user can belong to multiple teams (e.g., John is in both "Day Shift Team" and "Safety Committee")
- **One Team → Many Users**: A team contains multiple users (e.g., "Maintenance Team A" has 10 members)
- **Bridge Table**: `Team_Users` stores one row per user-team association

**Bridge Table Structure**:
```
Dim_Users_AssignedTo (1) ←→ (∞) Team_Users (∞) ←→ (1) Dim_Teams
```

**Role Indicators**:
The table includes two boolean flags that define each user's role within a team:
- **Is_Member**: Active membership status (true = currently on team, false = removed/inactive)
- **Is_Supervisor**: Supervisory authority within the team (true = supervisor, false = regular member)

**Possible Flag Combinations**:
| Is_Member | Is_Supervisor | Interpretation |
|-----------|---------------|----------------|
| TRUE | TRUE | Active team supervisor |
| TRUE | FALSE | Active regular team member |
| FALSE | TRUE | Former supervisor (historical record) |
| FALSE | FALSE | Former member (historical record) |

**User_TenantId Composite Key**:
The calculated column `User_TenantId` combines User_Id and Tenant_Id into a single string key:
```
User_TenantId = "12345_67"  // User ID 12345 in Tenant 67
```

This composite key:
- Provides a unique identifier across potential user ID collisions between tenants
- Enables single-column joins in scenarios where compound keys are challenging
- Maintains tenant isolation at the key level

**Historical Tracking**:
Unlike some bridge tables that only store current relationships, this table retains historical membership:
- `Is_Member = FALSE` indicates former team members (not deleted, just inactivated)
- `Created_Date` shows when user joined the team
- `Last_Updated` tracks membership status changes
- This enables historical team composition reporting (e.g., "Who was on Team A in Q1 2023?")

**Supervisor Pattern**:
The `Is_Supervisor` flag enables multiple supervisors per team:
- Teams can have 0, 1, or multiple supervisors
- Supervisors can supervise multiple teams
- Supervisor status is independent of member status (though typically `Is_Supervisor = TRUE` implies `Is_Member = TRUE`)

**Cross-Filtering Behavior**:
In standard bridge table patterns, cross-filtering is typically:
- `Dim_Users_AssignedTo → Team_Users`: Single direction (filter users → see their teams)
- `Team_Users → Dim_Teams`: Single direction (filter teams → see their members)
- Bidirectional cross-filtering would create filter ambiguity and is generally avoided

**Example Scenario - User with Multiple Team Memberships**:

User: John Smith (User_Id = 42)

Team Memberships in `Team_Users`:
| Team_Id | Team | Is_Member | Is_Supervisor | Created_Date |
|---------|------|-----------|---------------|--------------|
| 15 | Maintenance Team A | TRUE | FALSE | 2023-01-10 |
| 17 | Electrical Team | TRUE | TRUE | 2023-03-15 |
| 22 | Safety Committee | TRUE | FALSE | 2023-06-01 |
| 18 | Operations - Pit 3 | FALSE | FALSE | 2022-08-01 |

Interpretation:
- John is currently a **regular member** of Maintenance Team A (joined Jan 2023)
- John is a **supervisor** of the Electrical Team (promoted to supervisor Mar 2023)
- John is a **committee member** of the Safety Committee (added Jun 2023)
- John was **formerly part** of Operations - Pit 3 team (removed Aug 2022, historical record retained)

When filtering reports by John Smith:
- Assignments for any of his three active teams (15, 17, 22) appear
- Historical analysis can include his time on team 18

When filtering reports by Electrical Team:
- John appears as a supervisor
- His assignments and activities are included in team metrics

**Team Assignment Distribution**:
Bridge tables enable sophisticated assignment logic:
1. **Direct User Assignment**: Assignment → User (bypasses team)
2. **Team Assignment**: Assignment → Team → Team_Users → Users (distributed to team members)
3. **Supervisor Routing**: Assignment → Team → Team_Users (Is_Supervisor=TRUE) → Supervisor approvals

---

## Related Documentation
- **ERD_05_Users_Teams.md** - ERD diagram showing team-user relationship context
- **Dim_Teams.md** - Team dimension table
- **Dim_Users_AssignedTo.md** - User dimension table
- **Assignment_Details_Snapshot.md** - Historical team assignment tracking
- **Fact_Rosters.md** - Team shift scheduling

---

## Change History
| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-15 | Auto-generated | Initial documentation from TMDL metadata |

---

## Notes
- **Bridge Table Pattern**: This table implements the many-to-many bridge pattern between users and teams, enabling users to belong to multiple teams and teams to contain multiple users.
- **Boolean Flags**: The `Is_Member` and `Is_Supervisor` boolean columns define membership status and supervisory authority within teams.
- **Historical Retention**: Former team members are retained with `Is_Member = FALSE` rather than deleted, enabling historical team composition analysis.
- **Calculated Team Column**: The `Team` calculated column (`RELATED(Dim_Teams[Name])`) provides team name for easier filtering without joining to the team dimension.
- **Composite Key Pattern**: The `User_TenantId` calculated column combines User_Id and Tenant_Id into a single string identifier for scenarios requiring single-column keys.
- **Multiple Supervisors**: Teams can have multiple supervisors simultaneously (multiple rows with same Team_Id and Is_Supervisor = TRUE).
- **Supervisor Without Membership**: While typically supervisors are also members (Is_Supervisor = TRUE AND Is_Member = TRUE), the schema technically allows Is_Supervisor = TRUE with Is_Member = FALSE for historical supervisor records.
- **No Soft Deletes**: This table uses `Is_Member` flag for active/inactive status rather than a traditional IsDeleted column.
- **Tenant Isolation**: The table is filtered by Tenant_Id in Power Query, ensuring users only see team memberships within their tenant.
- **LastLoaded Timestamp**: The ETL timestamp enables data lineage tracking and refresh validation.
- **Cross-Filtering Direction**: Relationships from this bridge table to dimension tables should use single-direction cross-filtering to avoid filter ambiguity.
- **Team Calculated Column**: The DAX calculated column `Team = RELATED(Dim_Teams[Name])` duplicates data from the team dimension but improves usability in slicers and filters.
- **Created_Date Tracking**: The Created_Date timestamp tracks when users were added to teams, enabling tenure analysis and team history reporting.
- **Last_Updated Tracking**: The Last_Updated timestamp captures membership status changes (e.g., when Is_Member changed from TRUE to FALSE, or when promoted to supervisor).
- **No Effective Dates**: Unlike SCD Type 2 patterns, this table doesn't use effective-from/effective-to dates. Historical analysis requires filtering by Created_Date and Last_Updated timestamps.
