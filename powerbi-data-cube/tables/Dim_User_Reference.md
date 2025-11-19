# Dim_User_Reference

## Table Overview
`Dim_User_Reference` is a calculated table that provides a role-playing user dimension sourced from `Dim_Users_AssignedTo`. This table contains user reference data including identifiers, contact information, organizational structure, and role classification.

The table enables user-based filtering and grouping without creating ambiguous relationships when users appear in multiple contexts (assigned to, raised by, resolved by, etc.).

**Current Status**: Calculated table (refreshed when source `Dim_Users_AssignedTo` refreshes).

---

## Specifications
- **Source**: Calculated from `Dim_Users_AssignedTo` table
- **Row Count**: Same as source table (one row per user)
- **Grain**: One row per user
- **Primary Key**: `User_Id`
- **Incremental Refresh**: Not applicable (calculated table)
- **Partitioning Strategy**: Not applicable
- **Source Columns**: 12 (all from source table)
- **Calculated Columns**: 0 (beyond the table calculation itself)

---

## Column Specifications

| Column Name | Data Type | Format | Nullable | Hidden | Description |
|------------|-----------|--------|----------|--------|-------------|
| User_Id | Int64 | | No | No | Primary key for user identification |
| Email | String | | Yes | No | User's email address |
| Full_Name | String | | Yes | No | User's complete name (first + last) |
| Full_Name_Email | String | | Yes | No | Concatenated display format: "Full Name (email@domain.com)" |
| Role | String | | Yes | No | User's role classification (e.g., "Field Operator", "Supervisor", "Manager") |
| User_Code | String | | Yes | No | Short user identifier code |
| Reference_Code | String | | Yes | No | External system reference code for user |
| Authorisation_Code | String | | Yes | No | Authorization or permission level code |
| Department | String | | Yes | No | Department name user belongs to |
| Department_Code | String | | Yes | No | Department identifier code |
| Organisation | String | | Yes | No | Organization name user belongs to |
| Organisation_Code | String | | Yes | No | Organization identifier code |

---

## Calculated Columns
None. This is a calculated table that duplicates columns from `Dim_Users_AssignedTo`, but does not add calculated columns beyond the source structure.

---

## Relationships

### Outbound Relationships
None explicitly defined. This is a role-playing dimension that can be related to fact tables using different relationship paths.

### Inbound Relationships
Varies based on model configuration. Common relationships include:
- From `Assignment_FieldMeasurement_Exceptions` on `Raised_By` → `User_Id`
- From `Assignment_FieldMeasurement_Exceptions` on `Resolved_By` → `User_Id`
- From `Fact_User_LogOn_Activities` on `User_Id` → `User_Id`
- From `Fact_User_Assignment_Audit` on `User_Id` → `User_Id`
- From other tables requiring user context without conflicting with assigned-to relationships

---

## DAX Source

```dax
Dim_User_Reference = Dim_Users_AssignedTo
```

**Explanation**: This calculated table is a complete copy of `Dim_Users_AssignedTo`, creating a role-playing dimension for user contexts other than "assigned to" relationships.

---

## DAX Query Patterns

### Example 1: User Listing by Department
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_User_Reference[Department],
    Dim_User_Reference[Role],
    "User_Count", COUNTROWS(Dim_User_Reference),
    "Sample_Users", CONCATENATEX(
        TOPN(3, VALUES(Dim_User_Reference[Full_Name])),
        Dim_User_Reference[Full_Name],
        ", "
    )
)
ORDER BY Dim_User_Reference[Department], [User_Count] DESC
```

### Example 2: User Lookup by Email
```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Dim_User_Reference,
        "User_Id", Dim_User_Reference[User_Id],
        "Full_Name", Dim_User_Reference[Full_Name],
        "Email", Dim_User_Reference[Email],
        "Role", Dim_User_Reference[Role],
        "Department", Dim_User_Reference[Department],
        "Organisation", Dim_User_Reference[Organisation]
    ),
    SEARCH("@example.com", Dim_User_Reference[Email], 1, 0) > 0
)
```

### Example 3: User Reference Code Mapping
```dax
EVALUATE
SELECTCOLUMNS(
    FILTER(
        Dim_User_Reference,
        NOT ISBLANK(Dim_User_Reference[Reference_Code])
    ),
    "User_Id", Dim_User_Reference[User_Id],
    "Full_Name", Dim_User_Reference[Full_Name],
    "User_Code", Dim_User_Reference[User_Code],
    "Reference_Code", Dim_User_Reference[Reference_Code],
    "Authorisation_Code", Dim_User_Reference[Authorisation_Code]
)
ORDER BY Dim_User_Reference[Reference_Code]
```

### Example 4: Organization Structure Overview
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_User_Reference[Organisation],
    Dim_User_Reference[Organisation_Code],
    Dim_User_Reference[Department],
    Dim_User_Reference[Department_Code],
    "User_Count", COUNTROWS(Dim_User_Reference),
    "Roles", CONCATENATEX(
        VALUES(Dim_User_Reference[Role]),
        Dim_User_Reference[Role],
        ", "
    )
)
ORDER BY 
    Dim_User_Reference[Organisation],
    Dim_User_Reference[Department]
```

### Example 5: User Role Distribution
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_User_Reference[Role],
    "User_Count", COUNTROWS(Dim_User_Reference),
    "Departments", DISTINCTCOUNT(Dim_User_Reference[Department]),
    "Organisations", DISTINCTCOUNT(Dim_User_Reference[Organisation])
)
ORDER BY [User_Count] DESC
```

### Example 6: Find User by Partial Name
```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Dim_User_Reference,
        "User_Id", Dim_User_Reference[User_Id],
        "Full_Name_Email", Dim_User_Reference[Full_Name_Email],
        "Role", Dim_User_Reference[Role],
        "Department", Dim_User_Reference[Department],
        "User_Code", Dim_User_Reference[User_Code]
    ),
    SEARCH("Smith", Dim_User_Reference[Full_Name], 1, 0) > 0
)
ORDER BY Dim_User_Reference[Full_Name]
```

---

## Data Model Pattern

### Role-Playing Dimension Pattern
`Dim_User_Reference` implements the role-playing dimension pattern to solve the ambiguous relationship problem when users appear in multiple contexts within the same data model.

**Problem Scenario**:
In the Obzervr datacube, users can appear in multiple roles:
- Assignments have an "Assigned To" user (`Dim_Users_AssignedTo` relationship)
- Exceptions have both "Raised By" and "Resolved By" users
- Audit tables track "Performed By" users
- Roster tables reference scheduled users
- Logon activity tracks authenticated users

Creating direct relationships from all these contexts to a single `Dim_Users` table would create ambiguous or circular relationship paths.

**Solution**:
Create a calculated table (`Dim_User_Reference`) that duplicates the user dimension structure. This allows:
1. `Dim_Users_AssignedTo` handles "assigned to" relationships
2. `Dim_User_Reference` handles all other user contexts (raised by, resolved by, performed by, etc.)
3. Both tables contain identical data but serve different relationship roles

**Relationship Strategy**:
```
Assignments → [Assignment_Id → Assigned_To] → Dim_Users_AssignedTo
Assignment_FieldMeasurement_Exceptions → [Raised_By] → Dim_User_Reference
Assignment_FieldMeasurement_Exceptions → [Resolved_By] → Dim_User_Reference (inactive)
Fact_User_LogOn_Activities → [User_Id] → Dim_User_Reference
```

**Benefits**:
- **Eliminates Ambiguity**: Each user context has a clear, dedicated relationship path
- **Consistent Data**: Both tables contain identical user information
- **Simplified DAX**: Measures don't require complex context switching with USERELATIONSHIP
- **Maintainability**: Single source (`Dim_Users_AssignedTo`) automatically populates role-playing copy

**Trade-offs**:
- **Duplication**: User data exists in two table instances (though as a calculated table, storage impact is minimal)
- **Refresh Dependency**: Changes to `Dim_Users_AssignedTo` require calculated table refresh
- **Relationship Management**: Must carefully document which table serves which context

**Full_Name_Email Pattern**:
The `Full_Name_Email` column provides a user-friendly display format combining name and email:
```
"John Smith (john.smith@example.com)"
```
This is particularly useful in:
- Slicers and filters (users can search by name or email)
- Visual labels (clear identification without separate columns)
- Tooltips and drill-throughs

**Organizational Hierarchy**:
The table includes three levels of organizational structure:
1. **Organisation / Organisation_Code**: Top-level organization (e.g., "ABC Mining Corp", "ORG-001")
2. **Department / Department_Code**: Department within organization (e.g., "Operations", "DEPT-OPS")
3. **Role**: Job function within department (e.g., "Field Operator", "Supervisor")

This structure enables:
- Department-level filtering and reporting
- Organization-wide analytics
- Role-based access control (RLS) when configured

**Example Scenario - Exception Tracking**:

Without role-playing dimensions:
```
Assignment_FieldMeasurement_Exceptions → Dim_Users
(Ambiguous: Which relationship? Raised_By or Resolved_By?)
```

With role-playing dimensions:
```
Assignment_FieldMeasurement_Exceptions[Raised_By] → Dim_User_Reference[User_Id]
Assignment_FieldMeasurement_Exceptions[Resolved_By] → Dim_User_Reference[User_Id] (inactive)

// Measure for raised by user
Exceptions_Raised_By_User = COUNTROWS(Assignment_FieldMeasurement_Exceptions)

// Measure for resolved by user (uses inactive relationship)
Exceptions_Resolved_By_User = 
CALCULATE(
    COUNTROWS(Assignment_FieldMeasurement_Exceptions),
    USERELATIONSHIP(
        Assignment_FieldMeasurement_Exceptions[Resolved_By],
        Dim_User_Reference[User_Id]
    )
)
```

---

## Related Documentation
- **ERD_05_Users_Teams.md** - ERD diagram showing user and team relationship context
- **Dim_Users_AssignedTo.md** - Source table for this calculated dimension
- **Assignment_FieldMeasurement_Exceptions.md** - Uses this dimension for exception tracking
- **Fact_User_LogOn_Activities.md** - Uses this dimension for user activity tracking
- **Fact_User_Assignment_Audit.md** - Uses this dimension for audit trail

---

## Change History
| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-15 | Auto-generated | Initial documentation from TMDL metadata |

---

## Notes
- **Calculated Table**: This is a calculated table defined as `Dim_User_Reference = Dim_Users_AssignedTo`, making it a complete copy of the source dimension.
- **Role-Playing Pattern**: Solves ambiguous relationship issues when users appear in multiple contexts (assigned to, raised by, resolved by, performed by, etc.).
- **No Incremental Refresh**: As a calculated table, this inherits refresh behavior from its source table (`Dim_Users_AssignedTo`). When source refreshes, this table refreshes.
- **Identical Structure**: All columns and data match `Dim_Users_AssignedTo` exactly. The separation is purely for relationship clarity.
- **Relationship Flexibility**: This table can have multiple relationships to the same fact table (e.g., Raised_By and Resolved_By), with one active and others inactive using USERELATIONSHIP.
- **User_Code vs Reference_Code**: `User_Code` is typically an internal identifier, while `Reference_Code` may link to external HR or payroll systems.
- **Authorisation_Code**: This column may be used for permission-level classification or integration with access control systems.
- **Full_Name_Email Format**: The concatenated format "(email)" suggests consistent formatting logic in the source table, likely generated in Power Query or the source system.
- **Organization Hierarchy**: The Org → Dept → Role hierarchy enables multi-level filtering and RLS configuration.
- **NULL Handling**: Many columns are nullable, indicating not all users have complete organizational metadata (e.g., contractors may lack department codes).
- **Email Uniqueness**: While `User_Id` is the primary key, email addresses are likely unique and can serve as alternate lookup keys.
- **No Calculated Columns**: Beyond the table-level calculation (copying from source), this table doesn't add calculated columns. All column logic exists in the source table.
- **Refresh Performance**: As a calculated table, this has minimal refresh overhead - it simply points to the source table's data without additional transformations.
