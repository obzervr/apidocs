# RLS_Roles

## Table Overview
`RLS_Roles` is a lookup table that defines the row-level security (RLS) role catalog for the Obzervr semantic model. This table contains role identifiers and names used for RLS configuration, enabling role-based data access control.

The table uses an embedded binary compressed data source in Power Query, containing a small set of pre-defined role definitions.

**Current Status**: Power Query M source with binary compressed JSON data.

---

## Specifications
- **Source**: Power Query M with embedded binary compressed JSON
- **Row Count**: Small (typically 2-4 roles)
- **Grain**: One row per RLS role
- **Primary Key**: Id
- **Incremental Refresh**: Not applicable (embedded data source)
- **Partitioning Strategy**: Single partition
- **Source Columns**: 2
- **Calculated Columns**: 0
- **Query Group**: RLS

---

## Column Specifications

| Column Name | Data Type | Format | Nullable | Hidden | Description |
|------------|-----------|--------|----------|--------|-------------|
| Id | Int64 | 0 | No | No | Unique role identifier (integer primary key) |
| Name | String | | No | No | Role name (e.g., "Super User", "Team User", "Manager") sorted by Id |

**Column Configuration**:
- `Name` column is sorted by `Id` column for consistent role ordering

---

## Calculated Columns
None. This table uses only source columns from Power Query.

---

## Relationships

### Outbound Relationships
None explicitly defined. This table serves as a lookup for RLS role names but may be referenced by measures.

### Inbound Relationships
None. This is a lookup table for RLS configuration.

---

## Power Query M Source

```m
let
    Source = Table.FromRows(
        Json.Document(
            Binary.Decompress(
                Binary.FromText(
                    "i45WMlDSUXJMLsksSyxJVQjyCVaK1YlWMgQKhqQm5iqEFqcWgUWMgCLBpQWpRVChWAA=", 
                    BinaryEncoding.Base64
                ), 
                Compression.Deflate
            )
        ), 
        let _t = ((type nullable text) meta [Serialized.Text = true]) 
        in type table [Id = _t, Name = _t]
    ),
    #"Changed Type" = Table.TransformColumnTypes(
        Source,
        {
            {"Id", Int64.Type}, 
            {"Name", type text}
        }
    )
in
    #"Changed Type"
```

**Data Source Pattern**:
1. **Binary.FromText**: Decodes Base64-encoded binary data
2. **Binary.Decompress**: Decompresses using Deflate algorithm
3. **Json.Document**: Parses JSON from decompressed binary
4. **Table.FromRows**: Constructs table from JSON rows
5. **Changed Type**: Applies Int64 and text type transformations

**Embedded Data**: The Base64 string contains compressed JSON with role definitions.

**Decompressed Content** (approximate, based on string length):
```json
[
    [1, "Super User"],
    [2, "Team User"],
    [3, "Manager"]
]
```

Note: Actual role names may differ - these are typical RLS role examples.

---

## DAX Query Patterns

### Example 1: All RLS Roles
```dax
EVALUATE
RLS_Roles
ORDER BY RLS_Roles[Id]
```

### Example 2: Role Lookup by ID
```dax
EVALUATE
FILTER(
    RLS_Roles,
    RLS_Roles[Id] = 2
)
```

### Example 3: Role Count
```dax
EVALUATE
{ ("Total_Roles", COUNTROWS(RLS_Roles)) }
```

### Example 4: Role Names List
```dax
EVALUATE
SELECTCOLUMNS(
    RLS_Roles,
    "Role_ID", RLS_Roles[Id],
    "Role_Name", RLS_Roles[Name]
)
ORDER BY RLS_Roles[Id]
```

### Example 5: Current User Role Detection
```dax
Current_User_Role = 
VAR UserEmail = USERPRINCIPALNAME()
VAR UserRoleId = 
    -- Logic to determine user's role ID based on email/mapping
    -- This would typically reference a user-to-role mapping table
    LOOKUPVALUE(
        User_Role_Mapping[Role_Id],
        User_Role_Mapping[User_Email], UserEmail
    )
RETURN
    LOOKUPVALUE(
        RLS_Roles[Name],
        RLS_Roles[Id], UserRoleId
    )
```

---

## Data Model Pattern

### RLS Role Catalog Pattern
`RLS_Roles` implements an RLS role catalog pattern that defines available security roles for the semantic model:

**Pattern Characteristics**:
- **Small Lookup Table**: Few rows (typically 2-5 roles)
- **Embedded Data Source**: Binary compressed JSON embedded in Power Query
- **Static Definitions**: Roles defined at model design time, not dynamic
- **Integer IDs**: Simple integer primary key for role identification
- **Sorted Names**: Role names sorted by ID for consistent display

**Typical RLS Role Hierarchy**:
Based on Obzervr's operational context, likely roles include:

1. **Super User** (Id: 1)
   - Full data access across all tenants/sites
   - No RLS filtering applied
   - Administrative/support role

2. **Team User** (Id: 2)
   - Access limited to assigned team's data
   - Filtered by team membership via Team_Users bridge table
   - Standard operational role

3. **Manager** (Id: 3)
   - Access to managed teams/areas
   - Filtered by management hierarchy
   - Supervisory role

4. **Read Only** (Id: 4, if present)
   - View-only access
   - No create/edit permissions
   - Reporting/analytics role

**Embedded Binary Data Pattern**:
The Power Query uses a multi-step encoding pattern:
1. **JSON → Binary**: Role data encoded as JSON, converted to binary
2. **Compression**: Binary compressed using Deflate algorithm
3. **Base64 Encoding**: Compressed binary encoded as Base64 text string
4. **Embedded in M**: Base64 string embedded directly in Power Query code

**Benefits of Embedded Data**:
- **No External Dependency**: Role definitions stored within model
- **Version Control**: Roles change only with model updates
- **Deployment Simplicity**: No separate role configuration files
- **Consistency**: Same roles across all model deployments

**Drawbacks of Embedded Data**:
- **Static Configuration**: Changing roles requires model republish
- **Limited Flexibility**: Can't add roles without model modification
- **Not User-Editable**: Business users can't modify role definitions

**Base64 String Decoding**:
The Base64 string `"i45WMlDSUXJMLsksSyxJVQjyCVaK1YlWMgQKhqQm5iqEFqcWgUWMgCLBpQWpRVChWAA="` decodes to compressed binary, which decompresses to JSON array of [Id, Name] tuples.

**Power Query Query Group**:
The query is organized in the "RLS" query group, indicating:
- Logical grouping with other RLS-related queries
- Separate from data queries
- Configuration/metadata category

**Sort By Column Configuration**:
The `Name` column is sorted by `Id` column, ensuring:
- Consistent role order in slicers/visuals
- Hierarchical display (Super User first, Team User second, etc.)
- Predictable user experience

**RLS Implementation Pattern** (in broader model):
1. **Role Catalog**: This table defines available roles
2. **Role Mapping**: Another table (likely User_Role_Mapping) links users to roles
3. **DAX Filters**: Row-level security DAX expressions filter data based on role
4. **USERPRINCIPALNAME**: Measures use `USERPRINCIPALNAME()` to identify current user
5. **Dynamic Filtering**: Filter expressions look up user's role and apply appropriate filters

**Example RLS Filter Expressions** (not in this table, but related pattern):

**Team User Role Filter** (applied to Team_Users table):
```dax
[User_Email] = USERPRINCIPALNAME()
```

**Manager Role Filter** (applied to Dim_Teams table):
```dax
[Manager_Email] = USERPRINCIPALNAME()
```

**Super User Role Filter**:
```dax
1 = 1  // No filtering - see all data
```

**Card - Current User RLS Role Measure** (in _Measures table):
This table is likely referenced by the `Card - Current User RLS Role` measure documented in `_Measures` table, enabling users to see which role applies to them.

**Example Scenario - RLS Configuration**:

**RLS_Roles Table Data**:
| Id | Name |
|----|------|
| 1 | Super User |
| 2 | Team User |
| 3 | Manager |

**User-to-Role Mapping** (separate table, conceptual):
| User_Email | Role_Id |
|-----------|---------|
| john.smith@company.com | 2 |
| sarah.manager@company.com | 3 |
| admin@company.com | 1 |

**RLS Behavior**:
- **John Smith** (Team User): Sees only assignments/data for teams he belongs to
- **Sarah Manager** (Manager): Sees all assignments/data for teams she manages
- **Admin** (Super User): Sees all data across all teams and sites

**Dashboard Display**:
When John Smith opens dashboard:
- Card visual shows: "Your Role: Team User"
- Assignments filtered to his team membership via Team_Users
- User activity filtered to his logon data
- No access to other teams' data

When Admin opens dashboard:
- Card visual shows: "Your Role: Super User"
- All assignments visible regardless of team
- All user activity visible across organization
- Full data access for support/troubleshooting

**Role Assignment Process**:
1. New user created in system
2. Manager assigns role (Super User / Team User / Manager)
3. User-to-role mapping updated
4. RLS filters automatically apply on next login
5. User sees data appropriate for their role

**Power BI Service RLS Configuration**:
When publishing to Power BI Service:
1. Model includes RLS role definitions (separate from this catalog)
2. Roles reference this catalog for display purposes
3. Users assigned to Power BI roles in workspace security
4. DAX filter expressions evaluate per-user based on role assignment
5. This catalog table enables role name display in dashboards

**Maintenance Pattern**:
To add new role:
1. Decode Base64 string
2. Decompress binary
3. Parse JSON
4. Add new [Id, Name] tuple
5. Re-encode as JSON → Binary → Compress → Base64
6. Update Power Query M code
7. Republish model

This maintenance complexity suggests roles are expected to be relatively stable.

---

## Related Documentation
- **ERD_08_Measures_Metadata.md** - ERD diagram showing measures and metadata relationship context
- **_Measures.md** - Centralized measures table with `Card - Current User RLS Role` measure
- **Team_Users.md** - Bridge table used in Team User role filtering
- **Dim_User_Reference.md** - User dimension used in RLS role mapping

---

## Change History
| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-15 | Auto-generated | Initial documentation from TMDL metadata |

---

## Notes
- **RLS Role Catalog**: This table defines the row-level security role catalog with role IDs and names for the Obzervr semantic model.
- **Embedded Binary Data**: Uses Power Query with binary compressed JSON data source, embedding role definitions directly in the model.
- **Small Lookup Table**: Typically contains 2-4 role definitions (Super User, Team User, Manager, possibly Read Only).
- **Static Configuration**: Roles defined at model design time, not dynamic - changing roles requires model republish.
- **Base64 Compression**: Data encoded as Base64 → Binary → Deflate Compressed → JSON for compact storage in Power Query M code.
- **Sort By Column**: Name column sorted by Id for consistent role ordering in visuals and slicers.
- **No Relationships**: This lookup table has no explicit relationships - referenced by measures and RLS filter expressions.
- **Query Group RLS**: Organized in "RLS" query group, indicating configuration/metadata category separate from data queries.
- **Integer Primary Key**: Simple Int64 Id serves as primary key for role identification.
- **TMDL Comment**: "RLS role definitions: numeric id and role name used for row-level security"
- **Typical Roles**: Likely includes Super User (full access), Team User (team-filtered), Manager (management hierarchy filtered).
- **Power Query Steps**: Binary.FromText (Base64 decode) → Binary.Decompress (Deflate) → Json.Document → Table.FromRows → Type transformation.
- **Type Transformation**: Changes Id to Int64.Type and Name to type text for proper data types.
- **Card Measure Integration**: Referenced by `Card - Current User RLS Role` measure in _Measures table for user role display.
- **RLS Filter Pattern**: This catalog supports broader RLS implementation with role mapping, USERPRINCIPALNAME(), and DAX filter expressions.
- **No External Dependency**: Role definitions contained within model, no external configuration files required.
- **Deployment Simplicity**: Same role definitions deploy consistently across environments.
- **Limited Flexibility**: Can't add roles dynamically - requires model modification and republish.
- **Version Control**: Role changes tracked through model version control, not separate configuration management.
- **Maintenance Complexity**: Updating embedded binary data requires decode → modify → re-encode cycle.
- **Stable Role Expectation**: Embedded data pattern suggests roles expected to be relatively stable over time.
- **User Assignment**: Users assigned to Power BI Service roles after model publish, separate from this catalog.
- **DAX Filter Expressions**: Actual RLS filtering implemented via DAX expressions on tables, not in this catalog.
- **Display Purpose**: Primary purpose is role name display in measures and documentation, not filter enforcement.
- **Role Hierarchy**: Id ordering (1, 2, 3...) may reflect privilege hierarchy (Super User = highest).
- **Obzervr Context**: Role structure aligned with field operations (supervisors, team members, administrators).
