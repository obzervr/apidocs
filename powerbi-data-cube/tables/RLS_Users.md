# RLS_Users

**Table Type:** Security Table (All columns hidden)

**Purpose:** Filtered user list for row-level security implementation. Contains active users with tenant associations and analytics access flags for security filtering.

**Last Updated:** November 19, 2025

---

## Overview

RLS_Users is a security table that provides filtered active user data for implementing row-level security (RLS) across the semantic model. All columns are hidden from report consumers, and the table serves exclusively as a security filter context provider. The table includes analytics access flags derived from tenant-specific permissions, enabling granular control over which users can access analytics features for specific tenants.

---

## Columns

| Column Name | Data Type | Description | Hidden | Key |
|------------|-----------|-------------|--------|-----|
| **Unique_User_Id** | String | Unique identifier for the user in RLS mappings | Yes | - |
| **Email** | String | User email address | Yes | - |
| **Is_Active** | Boolean | Flag indicating whether the user is active | Yes | - |
| **Tenant_Id** | String | Tenant identifier for the user | Yes | FK |
| **User_Id** | String | Internal user identifier | Yes | FK |
| **Last_Updated** | DateTime | Last update timestamp for this RLS user record | Yes | - |
| **Full_Name** | String | Full display name for the user | Yes | - |
| **User_Tenant_Id** | String | Concatenated user and tenant identifier | Yes | - |
| **Has_Tenant_Analytics** | Integer | Indicator whether the tenant has analytics enabled (0 or 1) | Yes | - |

---

## Key Characteristics

### Security Table Pattern
- **All Hidden Columns:** Every column in this table is hidden to prevent direct access from reports
- **Filter Context Only:** Used exclusively in RLS filter expressions and security rules
- **No Direct Querying:** Report consumers cannot directly reference this table in visuals

### Active User Filtering
- Source query filters for `Is_Active = TRUE` only
- Ensures inactive/disabled users are excluded from security context
- Maintains current user roster for access control

### Multi-Tenant Support
- Each user record includes Tenant_Id for multi-tenant isolation
- User_Tenant_Id composite key format: `{User_Id}-{Tenant_Id}`
- Enables users to have different access levels across tenants

### Analytics Access Control
- **Has_Tenant_Analytics:** Derived from left outer join to Tenant_Analytic_Users table
- Value 0: User does NOT have analytics access for this tenant
- Value 1: User HAS analytics access for this tenant
- Controls visibility of advanced analytics features and reports

---

## Relationships

### Outgoing Relationships

**To Team_Users** (Many-to-Many)
- **Relationship ID:** f63b5255-18bb-7fe9-6b65-7eef23f72204
- **From Column:** RLS_Users[User_Id]
- **To Column:** Team_Users[User_Id]
- **Cardinality:** Many-to-Many (To-many cardinality)
- **Purpose:** Many-to-many bridge relationship for RLS filtering. Users can belong to multiple teams, and RLS_Users provides the security filter context.

---

## Data Source

**Source Type:** SQL Query

**Source Table/View:** DimUsers

**Transformation Logic:**
```sql
SELECT 
    UserId,
    Email,
    IsActive,
    TenantId,
    -- Additional columns
FROM DimUsers
LEFT OUTER JOIN Tenant_Analytic_Users ON [join condition]
WHERE IsActive = 1
    AND ([Tenant filtering logic])
```

**Key Transformations:**
1. Filter for active users only (IsActive = 1)
2. Left outer join to Tenant_Analytic_Users to determine Has_Tenant_Analytics flag
3. User_Tenant_Id calculated as composite key
4. Tenant filtering based on AllTenants or specific TenantId1-5 parameters

---

## Usage in Row-Level Security

### RLS Filter Expressions

**Example 1: Check if current user exists in RLS_Users**
```dax
// RLS Filter on Assignments table
VAR CurrentUser = USERPRINCIPALNAME()
VAR UserExists = 
    COUNTROWS(
        FILTER(
            RLS_Users,
            RLS_Users[Email] = CurrentUser
        )
    ) > 0
RETURN UserExists
```

**Example 2: Filter by analytics access**
```dax
// RLS Filter on analytics-specific tables
VAR CurrentUser = USERPRINCIPALNAME()
VAR HasAnalytics = 
    CALCULATE(
        MAX(RLS_Users[Has_Tenant_Analytics]),
        RLS_Users[Email] = CurrentUser
    )
RETURN HasAnalytics = 1
```

**Example 3: Team-based filtering via relationship**
```dax
// RLS Filter using Team_Users relationship
VAR CurrentUser = USERPRINCIPALNAME()
VAR UserTeams = 
    CALCULATETABLE(
        VALUES(Team_Users[Team_Id]),
        RLS_Users[Email] = CurrentUser
    )
RETURN
    Assignments[Team_Id] IN UserTeams
```

---

## Common DAX Patterns

### Check Current User Analytics Access
```dax
Current User Has Analytics = 
VAR CurrentUser = USERPRINCIPALNAME()
VAR AnalyticsAccess = 
    CALCULATE(
        MAX(RLS_Users[Has_Tenant_Analytics]),
        RLS_Users[Email] = CurrentUser
    )
RETURN
    IF(ISBLANK(AnalyticsAccess), 0, AnalyticsAccess)
```

### Get Current User's Tenants
```dax
Current User Tenant List = 
VAR CurrentUser = USERPRINCIPALNAME()
RETURN
    CONCATENATEX(
        FILTER(RLS_Users, RLS_Users[Email] = CurrentUser),
        RLS_Users[Tenant_Id],
        ", "
    )
```

### Count Active RLS Users
```dax
Active RLS User Count = 
CALCULATE(
    DISTINCTCOUNT(RLS_Users[User_Id]),
    RLS_Users[Is_Active] = TRUE()
)
```

---

## Related Tables

### Direct Relationships
- **Team_Users** - Many-to-many bridge for team membership filtering

### Related Security Tables
- **RLS_Roles** - Role definitions used in security expressions
- **RLS_Tenant_User_Analytic** - Derived table filtered to users with analytics access
- **Dim_User_Reference** - User attribute dimension (not directly related but semantically linked)

### Referenced in RLS on These Tables
- Assignments
- TimeSeries
- TimeSeries_FieldMeasurements
- Fact_User_LogOn_Activities
- Fact_User_Audit_Command_Count_By_Day
- All tenant-specific fact and dimension tables

---

## Security Considerations

### Best Practices
1. **Never Unhide Columns:** Keep all columns hidden to maintain security boundary
2. **Use USERPRINCIPALNAME():** Always compare against current user's email
3. **Handle Missing Users:** Include logic for users not found in RLS_Users
4. **Test RLS Rules:** Use "View as Role" feature to validate security filtering
5. **Regular User List Updates:** Ensure RLS_Users refresh schedule keeps pace with user changes

### Performance Optimization
- Index User_Id and Email columns at source database level
- Consider incremental refresh if user table grows large (currently likely full refresh)
- Minimize complex calculations in RLS filter expressions
- Use CALCULATE with simple filter predicates where possible

### Audit and Monitoring
- Track Last_Updated timestamps to identify stale user records
- Monitor Has_Tenant_Analytics flag changes for access control auditing
- Log RLS rule failures or unexpected blank results
- Validate User_Tenant_Id uniqueness regularly

---

## Data Model Position

**Related ERDs:**
- **ERD #5: User, Team & Security** - Primary documentation
- **ERD #1: Assignment Core Model** - RLS filtering for Assignments
- **ERD #7: Fact Tables & Audit** - RLS filtering for audit fact tables

**Model Layer:** Security Infrastructure

**Refresh Frequency:** Should align with user provisioning processes (typically daily or more frequent)

---

## Change History

| Date | Change Description | Modified By |
|------|-------------------|-------------|
| 2025-11-19 | Initial table documentation created | AI Documentation Generator |

---

## Additional Notes

### Analytics Access Logic
The Has_Tenant_Analytics flag is derived from a left outer join to a separate Tenant_Analytic_Users table (not fully documented). This pattern allows:
- Centralized management of analytics permissions
- Tenant-specific access control separate from basic RLS
- Ability to grant/revoke analytics access without modifying core user records

### Composite Key Pattern
The User_Tenant_Id column follows the pattern `{User_Id}-{Tenant_Id}`, enabling:
- Unique identification of user-tenant combinations
- Cross-filtering in multi-tenant scenarios
- Simplified relationship joins where composite keys are needed

### Hidden Table Benefits
Marking all columns as hidden:
- Prevents accidental exposure of security-sensitive data
- Ensures RLS logic remains centralized in defined rules
- Reduces cognitive load for report developers (table not visible in field list)
- Maintains clean separation between data model and security model
