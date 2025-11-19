# RLS_Tenant_User_Analytic

**Table Type:** Security Bridge Table (Hidden)

**Purpose:** Filtered bridge table identifying users with analytics access for specific tenants. Derived from RLS_Users filtered to Has_Tenant_Analytics = 1.

**Last Updated:** November 19, 2025

---

## Overview

RLS_Tenant_User_Analytic is a security bridge table that provides a filtered subset of RLS_Users containing only user-tenant combinations where the user has been granted analytics access. This hidden table serves as a specialized filter context for analytics-specific reports and features, enabling granular control over which users can view advanced analytics for which tenants.

The table is derived entirely from RLS_Users through Power Query filtering, making it a calculated/filtered reference table rather than an independent data source.

---

## Columns

| Column Name | Data Type | Description | Hidden | Key |
|------------|-----------|-------------|--------|-----|
| **User_Id** | String | User identifier | Yes | FK |
| **Tenant_Id** | String | Tenant identifier | Yes | FK |
| **Tenant_Code** | String | Tenant code (calculated via LOOKUPVALUE from Dim_Tenant) | Yes | - |

---

## Key Characteristics

### Filtered Reference Table
- **Source:** Power Query reference to RLS_Users
- **Filter Condition:** `Has_Tenant_Analytics = 1`
- **Reduced Columns:** Only User_Id, Tenant_Id retained; other columns removed
- **Purpose:** Provides clean analytics access list without security noise

### Hidden Security Table
- **IsHidden:** True (entire table hidden from report consumers)
- **Access Method:** Used exclusively via inactive relationships activated with USERELATIONSHIP()
- **No Direct Usage:** Not accessible in field lists or visual configurations

### Calculated Tenant_Code Column
```dax
Tenant_Code = 
LOOKUPVALUE(
    Dim_Tenant[Tenant_Code],
    Dim_Tenant[Tenant_Id],
    RLS_Tenant_User_Analytic[Tenant_Id]
)
```
- Provides human-readable tenant identification
- Useful in DAX debugging and security expression validation

### Multi-Tenant Analytics Granularity
- Users can have analytics access to some tenants but not others
- Each row represents one user-tenant analytics permission
- Supports complex organizational structures with mixed access levels

---

## Relationships

### Incoming Relationships

**From Team_Users** (Inactive, Many-to-Many, Bidirectional)
- **Relationship ID:** 63d8ef54-8763-6b2b-8fc9-5f1f2016bd48
- **From Column:** Team_Users[Tenant_Id]
- **To Column:** RLS_Tenant_User_Analytic[Tenant_Id]
- **Cardinality:** Many-to-Many (To-many cardinality)
- **Cross-Filtering:** Both Directions
- **Active:** False (must use USERELATIONSHIP to activate)
- **Purpose:** Inactive relationship used with USERELATIONSHIP in DAX for analytics access control. Filters team membership to users with tenant analytics privileges.

---

## Data Source

**Source Type:** Power Query Reference

**Source Table:** RLS_Users

**Transformation Logic:**
```m
let
    Source = RLS_Users,
    #"Filtered Rows" = Table.SelectRows(Source, each [Has_Tenant_Analytics] = 1),
    #"Removed Other Columns" = Table.SelectColumns(#"Filtered Rows", {"User_Id", "Tenant_Id"}),
    #"Added Tenant_Code" = Table.AddColumn(#"Removed Other Columns", "Tenant_Code", 
        each [calculated LOOKUPVALUE logic])
in
    #"Added Tenant_Code"
```

**Key Steps:**
1. Reference RLS_Users table
2. Filter where Has_Tenant_Analytics = 1
3. Remove unnecessary columns (keep only User_Id, Tenant_Id)
4. Add calculated Tenant_Code column via LOOKUPVALUE

---

## Usage in Row-Level Security

### Activating the Inactive Relationship

**Pattern:** Use USERELATIONSHIP() in DAX measures to activate the analytics access filter

**Example 1: Filter assignments by analytics access**
```dax
Assignments with Analytics Access = 
CALCULATE(
    COUNTROWS(Assignments),
    USERELATIONSHIP(Team_Users[Tenant_Id], RLS_Tenant_User_Analytic[Tenant_Id])
)
```

**Example 2: Check if user has analytics access for current tenant**
```dax
Has Analytics Access = 
VAR CurrentUser = USERPRINCIPALNAME()
VAR CurrentTenant = SELECTEDVALUE(Dim_Tenant[Tenant_Id])
VAR UserTenants = 
    CALCULATETABLE(
        VALUES(RLS_Tenant_User_Analytic[Tenant_Id]),
        USERELATIONSHIP(Team_Users[Tenant_Id], RLS_Tenant_User_Analytic[Tenant_Id]),
        FILTER(RLS_Users, RLS_Users[Email] = CurrentUser)
    )
RETURN
    CurrentTenant IN UserTenants
```

**Example 3: RLS rule for analytics-specific reports**
```dax
// Apply to report page or entire semantic model
VAR CurrentUser = USERPRINCIPALNAME()
VAR AnalyticTenants = 
    CALCULATETABLE(
        VALUES(RLS_Tenant_User_Analytic[Tenant_Id]),
        RLS_Users[Email] = CurrentUser
    )
RETURN
    Dim_Tenant[Tenant_Id] IN AnalyticTenants
```

---

## Common DAX Patterns

### List Current User's Analytic Tenants
```dax
Current User Analytic Tenants = 
VAR CurrentUser = USERPRINCIPALNAME()
RETURN
    CONCATENATEX(
        FILTER(
            RLS_Tenant_User_Analytic,
            RELATED(RLS_Users[Email]) = CurrentUser
        ),
        RLS_Tenant_User_Analytic[Tenant_Code],
        ", ",
        RLS_Tenant_User_Analytic[Tenant_Code],
        ASC
    )
```

### Count Users with Analytics Access per Tenant
```dax
Analytics Users per Tenant = 
CALCULATE(
    DISTINCTCOUNT(RLS_Tenant_User_Analytic[User_Id]),
    ALLEXCEPT(RLS_Tenant_User_Analytic, RLS_Tenant_User_Analytic[Tenant_Id])
)
```

### Check Analytics Access Boolean
```dax
User Has Analytics = 
VAR CurrentUser = USERPRINCIPALNAME()
VAR HasAccess = 
    CALCULATE(
        COUNTROWS(RLS_Tenant_User_Analytic),
        FILTER(
            RLS_Users,
            RLS_Users[Email] = CurrentUser
        )
    ) > 0
RETURN HasAccess
```

### Filter Fact Table by Analytics Access
```dax
Fact Rows with Analytics = 
CALCULATE(
    COUNTROWS(FactTable),
    USERELATIONSHIP(Team_Users[Tenant_Id], RLS_Tenant_User_Analytic[Tenant_Id]),
    RLS_Users[Email] = USERPRINCIPALNAME()
)
```

---

## Related Tables

### Source Table
- **RLS_Users** - Parent table from which this table is derived

### Relationship Tables
- **Team_Users** - Bridge table with inactive relationship for analytics filtering
- **Dim_Tenant** - Referenced via LOOKUPVALUE for Tenant_Code calculation

### Semantically Related
- **RLS_Roles** - Role definitions that may grant analytics permissions
- **Dim_User_Reference** - User attributes (connected via User_Id)

---

## Why Use an Inactive Relationship?

### Design Rationale
The relationship from Team_Users to RLS_Tenant_User_Analytic is intentionally inactive because:

1. **Ambiguity Avoidance:** Team_Users already has active relationship to Dim_Tenant via Tenant_Id
2. **Selective Activation:** Analytics filtering should only apply when explicitly requested
3. **Default Behavior:** Most reports should use standard tenant filtering, not analytics-only filtering
4. **Explicit Control:** USERELATIONSHIP makes analytics filtering obvious in DAX code
5. **Flexibility:** Different measures can use different tenant filtering contexts

### When to Activate
Activate this relationship when:
- Building analytics-specific reports or pages
- Filtering data to users with advanced permissions
- Implementing tiered access control (basic vs analytics users)
- Validating analytics access in security expressions

---

## Security Considerations

### Access Control Pattern
This table implements a **positive security model**:
- Only users explicitly granted analytics access appear in the table
- Missing entries mean NO analytics access (secure by default)
- Easier to audit than negative security (exclusion lists)

### Best Practices
1. **Keep Table Hidden:** Never expose to report consumers
2. **Document Usage:** Comment DAX measures that use USERELATIONSHIP
3. **Test Thoroughly:** Validate analytics access for various user scenarios
4. **Sync with RLS_Users:** Ensure refresh schedules align
5. **Monitor Changes:** Track additions/removals to analytics access list

### Performance Considerations
- Small table size (subset of RLS_Users) enables fast filtering
- Inactive relationship has no performance impact when not used
- USERELATIONSHIP may be slightly slower than active relationships (acceptable trade-off)
- Consider indexing User_Id and Tenant_Id at source if performance issues arise

---

## Data Model Position

**Related ERDs:**
- **ERD #5: User, Team & Security** - Primary documentation
- **ERD #7: Fact Tables & Audit** - Used for analytics-specific audit filtering

**Model Layer:** Security Infrastructure

**Refresh Dependency:** Depends on RLS_Users refresh (derived table)

**Refresh Strategy:** Refreshes automatically when RLS_Users refreshes (Power Query reference)

---

## Troubleshooting

### Common Issues

**Issue 1: User has Has_Tenant_Analytics = 1 but doesn't appear in this table**
- **Cause:** RLS_Users hasn't refreshed since permission granted
- **Solution:** Trigger manual refresh or wait for scheduled refresh

**Issue 2: USERELATIONSHIP not working as expected**
- **Cause:** Syntax error or wrong relationship referenced
- **Solution:** Verify relationship ID and columns; check for typos in USERELATIONSHIP function

**Issue 3: Tenant_Code showing BLANK()**
- **Cause:** Tenant_Id in RLS_Tenant_User_Analytic doesn't exist in Dim_Tenant
- **Solution:** Validate referential integrity; check for orphaned Tenant_Id values

**Issue 4: Performance degradation when using USERELATIONSHIP**
- **Cause:** Complex DAX expression or large data volume
- **Solution:** Simplify DAX; consider materializing analytics access in separate table

---

## Change History

| Date | Change Description | Modified By |
|------|-------------------|-------------|
| 2025-11-19 | Initial table documentation created | AI Documentation Generator |

---

## Additional Notes

### Derived Table Pattern
This table demonstrates the "derived table" pattern in Power BI:
- Reduces redundancy (no duplicate data storage)
- Maintains referential integrity automatically
- Simplifies maintenance (changes to RLS_Users flow through)
- Enables specialized filtering without source query complexity

### Analytics Access Hierarchy
Typical analytics access hierarchy:
1. **Super Users** (full access to all tenants and analytics)
2. **Analytics Users** (access to analytics for specific tenants) ← This table
3. **Team Users** (access to team-assigned data, no analytics)
4. **Guest Users** (limited read-only access)

This table specifically identifies users at level 2, enabling tiered security models.
