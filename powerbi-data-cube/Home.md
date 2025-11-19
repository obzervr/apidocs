# Obzervr Data Cube Documentation

Welcome to the Obzervr Power BI Data Cube documentation. This semantic model provides comprehensive analytics for assignment management, time tracking, field measurements, and operational reporting across 100+ tables organized into 8 logical domains.

---

## Quick Navigation

### 📊 [Entity Relationship Diagrams (ERDs)](#entity-relationship-diagrams)
Browse the data model organized by functional area with visual diagrams and relationship details.

### 📋 [Table Reference](#table-reference)
Complete alphabetical listing of all tables with links to detailed documentation.

### 🎯 [Getting Started](#getting-started)
Role-based guides for data engineers, report developers, and business analysts.

### 🔍 [Key Concepts](#key-concepts)
Understanding the core patterns and architecture of the data model.

---

## Entity Relationship Diagrams

The data model is organized into 8 functional ERDs. Each ERD document includes table descriptions, relationship diagrams, and DAX query patterns.

### [ERD #1: Assignment Core Model](ERDs/ERD_01_Assignment_Core.md)
**Tables:** 14 | **Focus:** Assignment lifecycle, locations, status, and categories

Core fact table `Assignments` with hierarchical location dimension (`Dim_AssignmentPoints`), status tracking, and category classification. Essential for work order and assignment reporting.

**Key Tables:** `Assignments`, `Dim_AssignmentPoints`, `Dim_Assignment_Status`, `Dim_Assignment_Categories`, `Dim_Sites`

---

### [ERD #2: Date Dimensions & Time Intelligence](ERDs/ERD_02_Date_Dimensions_Time_Intelligence.md)
**Tables:** 15 | **Focus:** Date dimensions and temporal analysis

Master date dimension with 6 role-playing contexts (FromDate, CompletedOn, FinalisedOn, ShiftDate, Exception), relative date filtering, and comprehensive time intelligence support.

**Key Tables:** `Dim_Date_Reference`, `Dim_Date_FromDate`, `Dim_Date_CompletedOn`, `Relative_Dates`, `Dim_Shifts`

---

### [ERD #3: Field Measurements & Time Series](ERDs/ERD_03_Field_Measurements_Time_Series.md)
**Tables:** 10 | **Focus:** Equipment hierarchies and measurement tracking

Time series framework with parent-child hierarchies for equipment groups, field measurement definitions, and document attachments. Supports dynamic attribute tracking via EAV pattern.

**Key Tables:** `TimeSeries`, `TimeSeries_FieldMeasurements`, `AssignmentPoint_Attributes`, `Dim_Shift_Time`

---

### [ERD #4: Assignment Details & Snapshots](ERDs/ERD_04_Assignment_Details_Snapshots.md)
**Tables:** 7 | **Focus:** Historical tracking and assignment attributes

Daily snapshots for trend analysis, assignment tagging, field operator tracking, and exception management. Implements SCD Type 2 pattern for historical reporting.

**Key Tables:** `Assignment_Details_Snapshot`, `Assignment_Completion_Percentage_Snapshot`, `Assignment_Tags`, `Assignment_FieldOperators`

---

### [ERD #5: User, Team & Security](ERDs/ERD_05_User_Team_Security.md)
**Tables:** 6 | **Focus:** User dimensions and row-level security

Master user dimension with 3 role-playing contexts (AssignedTo, CompletedBy, FinalisedBy), team membership, organizational hierarchy, and RLS support.

**Key Tables:** `Dim_User_Reference`, `Dim_Users_AssignedTo`, `Dim_Teams`, `Team_Users`

---

### [ERD #6: Templates, Fragments & Configuration](ERDs/ERD_06_Templates_Fragments_Configuration.md)
**Tables:** 10 | **Focus:** Work templates and fragment hierarchy

Work template library with 3-level fragment hierarchy (Groups → Sections → Fields), template grouping, and worklist view configuration.

**Key Tables:** `Dim_WorkTemplates`, `Dim_Fragments`, `Fact_Fragment_Details`, `Dim_Worklist_Views`

---

### [ERD #7: Fact Tables & Audit](ERDs/ERD_07_Fact_Tables_Audit.md)
**Tables:** 8 | **Focus:** Audit trails, usage analytics, and rosters

User activity logging, command audit trails, shift roster management, and operational fact tables. Supports incremental refresh for large audit tables.

**Key Tables:** `Fact_User_LogOn_Activities`, `Fact_User_Assignment_Audit`, `Fact_Rosters`, `Fact_Entity_Table_Contents`

---

### [ERD #8: Parameters, Lookups & Helpers](ERDs/ERD_08_Parameters_Lookups_Helpers.md)
**Tables:** 35+ | **Focus:** Parameters, lookups, and semantic model metadata

Parameter tables for dynamic selection, boolean helper dimensions, filtered dimensions, semantic model metadata, RLS support, and configuration tables. Includes centralized `_Measures` table with 200+ DAX measures.

**Key Tables:** `_Measures`, Parameter tables (7), Boolean helpers (5), Semantic model metadata (4), RLS tables (3)

---

## Table Reference

Browse all tables alphabetically with links to detailed documentation. Each table document includes column definitions, relationships, DAX patterns, and usage examples.

### Core Fact Tables
- [Assignments](tables/Assignments.md) - Central assignment/work order fact table (40 columns)
- [Fact_Fragment_Details](tables/Fact_Fragment_Details.md) - Fragment specifications (incremental refresh)
- [Fact_Rosters](tables/Fact_Rosters.md) - Shift roster planning records
- [Fact_User_Assignment_Audit](tables/Fact_User_Assignment_Audit.md) - Assignment audit trail (incremental refresh)
- [Fact_User_Audit_Command_Count_By_Day](tables/Fact_User_Audit_Command_Count_By_Day.md) - Daily command counts
- [Fact_User_LogOn_Activities](tables/Fact_User_LogOn_Activities.md) - User logon tracking

### Assignment Dimensions
- [Dim_AssignmentPoints](tables/Dim_AssignmentPoints.md) - Hierarchical location/asset dimension (Area → Location → Point)
- [Dim_Assignment_Categories](tables/Dim_Assignment_Categories.md) - Assignment categories
- [Dim_Assignment_Status](tables/Dim_Assignment_Status.md) - Assignment status values
- [Dim_Priority](tables/Dim_Priority.md) - Priority levels
- [Dim_Sites](tables/Dim_Sites.md) - Legacy site hierarchy (deprecated)
- [Dim_Subsites](tables/Dim_Subsites.md) - Legacy subsite hierarchy (deprecated)

### Date Dimensions
- [Dim_Date_Reference](tables/Dim_Date_Reference.md) - Master date dimension (2000-2050)
- [Dim_Date_CompletedOn](tables/Dim_Date_CompletedOn.md) - Role-playing: completion dates
- [Dim_Date_Exception](tables/Dim_Date_Exception.md) - Role-playing: exception dates
- [Dim_Date_FinalisedOn](tables/Dim_Date_FinalisedOn.md) - Role-playing: finalization dates
- [Dim_Date_FromDate](tables/Dim_Date_FromDate.md) - Role-playing: from dates
- [Dim_Date_ShiftDate](tables/Dim_Date_ShiftDate.md) - Role-playing: shift dates
- [Dim_Shifts](tables/Dim_Shifts.md) - Shift dimension with date keys
- [Relative_Dates](tables/Relative_Dates.md) - Bridge table for relative date filtering

### User & Team Dimensions
- [Dim_User_Reference](tables/Dim_User_Reference.md) - Master user dimension with org hierarchy
- [Dim_Users_AssignedTo](tables/Dim_Users_AssignedTo.md) - Role-playing: assignment owner
- [Dim_Users_Completed_By](tables/Dim_Users_Completed_By.md) - Role-playing: completer
- [Dim_Users_Finalised_By](tables/Dim_Users_Finalised_By.md) - Role-playing: finalizer
- [Dim_Teams](tables/Dim_Teams.md) - Team dimension
- [Team_Users](tables/Team_Users.md) - Many-to-many user-team membership

### Template & Fragment Tables
- [Dim_WorkTemplates](tables/Dim_WorkTemplates.md) - Work template definitions
- [Dim_Published_Worktemplates](tables/Dim_Published_Worktemplates.md) - Published templates (calculated)
- [Dim_Template_Groups](tables/Dim_Template_Groups.md) - Template grouping
- [Dim_Fragments](tables/Dim_Fragments.md) - Fragment hierarchy (Groups → Sections → Fields)
- [Dim_Worklist_Views](tables/Dim_Worklist_Views.md) - Worklist view configuration

### Time Series & Measurements
- [TimeSeries](tables/TimeSeries.md) - Equipment/series hierarchy with parent-child
- [TimeSeries_FieldMeasurements](tables/TimeSeries_FieldMeasurements.md) - Field measurement definitions
- [AssignmentPoint_Attributes](tables/AssignmentPoint_Attributes.md) - Dynamic attributes (EAV pattern)
- [Dim_Shift_Time](tables/Dim_Shift_Time.md) - Shift time dimension

### Assignment Detail Tables
- [Assignment_Details_Snapshot](tables/Assignment_Details_Snapshot.md) - Daily snapshots (SCD Type 2)
- [Assignment_Completion_Percentage_Snapshot](tables/Assignment_Completion_Percentage_Snapshot.md) - Completion tracking
- [Assignment_Tags](tables/Assignment_Tags.md) - Many-to-many tagging
- [Assignment_FieldOperators](tables/Assignment_FieldOperators.md) - Field operator assignments
- [Assignment_FieldMeasurement_Exceptions](tables/Assignment_FieldMeasurement_Exceptions.md) - Exception tracking

### Parameters & Helpers
- [_Measures](tables/_Measures.md) - Centralized measures table (200+ DAX measures)
- Parameter tables: [Asset_Hours_Type](tables/Parameter_Asset_Hours_Type.md), [Number_Measures](tables/Parameter_Number_Measures.md), [Previous_Period](tables/Parameter_Previous_Period.md), [Reading_Slicer](tables/Parameter_Reading_Slicer.md), [Referenc_Date_Type](tables/Parameter_Referenc_Date_Type.md), [Show_Cumulative](tables/Parameter_Show_Cumulative.md), [Show_Range](tables/Parameter_Show_Range.md)
- Boolean helpers: [Dim_Is_Completed](tables/Dim_Is_Completed.md), [Dim_Is_Assigned](tables/Dim_Is_Assigned.md), [Dim_Is_Outstanding](tables/Dim_Is_Outstanding.md), [Dim_Is_Resolved](tables/Dim_Is_Resolved.md), [Dim_Is_AssignmentPoint_Active](tables/Dim_Is_AssignmentPoint_Active.md)
- Semantic model metadata: [Semantic_Model_Tables](tables/Semantic_Model_Tables.md), [Semantic_Model_Columns](tables/Semantic_Model_Columns.md), [Semantic_Model_Relationships](tables/Semantic_Model_Relationships.md), [Semantic_Model_Measures](tables/Semantic_Model_Measures.md)

### Configuration & System
- [Dim_Tenant](tables/Dim_Tenant.md) - Multi-tenancy configuration
- [Last_Refreshed_UTC](tables/Last_Refreshed_UTC.md) - Dataset refresh timestamp
- [Organisation_Info](tables/Organisation_Info.md) - Organization branding
- [Version_Change_History](tables/Version_Change_History.md) - Version tracking

### RLS Tables
- [RLS_Roles](tables/RLS_Roles.md) - Role definitions
- [RLS_Users](tables/RLS_Users.md) - User-role mapping
- [RLS_Tenant_User_Analytic](tables/RLS_Tenant_User_Analytic.md) - User-tenant permissions

### Complete Table List
> 📁 **[Browse all tables in the tables folder →](tables/)**

---

## Getting Started

### For Report Developers

**1. Understand the Assignment Lifecycle**
- Start with [ERD #1: Assignment Core](ERDs/ERD_01_Assignment_Core.md) to learn the central `Assignments` fact table
- Review `Dim_Assignment_Status` for status values (Draft → Active → Completed → Finalised)
- Explore `Dim_AssignmentPoints` for the location hierarchy (Area → Location → Point)

**2. Master Time Intelligence**
- Review [ERD #2: Date Dimensions](ERDs/ERD_02_Date_Dimensions_Time_Intelligence.md) for date filtering patterns
- Use `Dim_Date_Reference` for standard date intelligence (YTD, MTD, etc.)
- Use `Relative_Dates` for user-friendly filters (Last 7 Days, This Month, etc.)
- Use role-playing dimensions for different date contexts (FromDate vs CompletedOn)

**3. Leverage Parameters**
- Check [ERD #8: Parameters & Helpers](ERDs/ERD_08_Parameters_Lookups_Helpers.md) for available parameters
- Use `Parameter_Referenc_Date_Type` to let users switch between FromDate, CompletedOn, and FinalisedOn
- Use Boolean helper dimensions (`Dim_Is_Completed`, etc.) for friendly filter labels

**4. Common DAX Patterns**
```dax
// Count assignments by status
Total Active Assignments = 
CALCULATE(
    COUNTROWS(Assignments),
    Dim_Assignment_Status[Status_Name] = "Active"
)

// Use role-playing dimension for completed assignments
Assignments Completed This Month = 
CALCULATE(
    COUNTROWS(Assignments),
    USERELATIONSHIP(Assignments[Completed_On_Date_Key], Dim_Date_CompletedOn[Date_Key]),
    Dim_Date_CompletedOn[Year_Month] = SELECTEDVALUE(Dim_Date_Reference[Year_Month])
)

// Relative date filtering
Assignments Last 7 Days = 
CALCULATE(
    COUNTROWS(Assignments),
    USERELATIONSHIP(Assignments[From_Date_Key], Relative_Dates[Date_Key]),
    Dim_Relative_Dates_Type[Type] = "Last 7 Days"
)
```

### For Data Engineers

**1. Understand the Source Structure**
- TMDL files located in: `Standard/Obzervr - Data Cube.SemanticModel/definition/`
- Review `relationships.tmdl` for relationship definitions
- Check individual table TMDL files in `tables/` folder

**2. Key Architecture Patterns**
- **Star Schema**: Central fact tables (`Assignments`, `Fact_Rosters`) surrounded by dimensions
- **Role-Playing Dimensions**: Date and User dimensions used in multiple contexts (inactive relationships)
- **Bridge Tables**: Many-to-many relationships (`Relative_Dates`, `Assignment_Tags`, `Team_Users`)
- **Soft Deletes**: `IsDeleted` flag pattern (no hard deletes)
- **Date Keys**: Integer YYYYMMDD format (e.g., 20251118)

**3. Multi-Tenancy**
- All tables filtered by `TenantId` via Power Query parameters
- `Dim_Tenant` table contains tenant configurations
- RLS implemented via `RLS_*` tables and `USERPRINCIPALNAME()`

**4. Incremental Refresh**
- `Fact_User_Assignment_Audit`: Filtered by Fired_Timestamp (2-5 year rolling window)
- `Fact_Fragment_Details`: Filtered by Last_Updated (5-year rolling window)

### For Business Analysts

**1. Common Analysis Scenarios**

**Assignment Status Funnel**
- Use `Assignments` with `Dim_Assignment_Status`
- Filter by date using `Dim_Date_FromDate`
- Drill down by `Dim_AssignmentPoints` hierarchy

**User Productivity**
- Use `Assignments` with `Dim_Users_Completed_By`
- Count by `CompletedOn` date
- Compare across teams via `Team_Users`

**Template Usage**
- Use `Assignments` with `Dim_WorkTemplates`
- Group by `Dim_Template_Groups`
- Review fragment details in `Fact_Fragment_Details`

**2. Understanding Filters**
- **Relative Dates**: "Last 7 Days", "This Month", "Last Quarter" (via `Relative_Dates`)
- **Boolean Helpers**: "Completed", "Outstanding", "Assigned" (via `Dim_Is_*` tables)
- **Date Context**: Switch between FromDate, CompletedOn, FinalisedOn (via `Parameter_Referenc_Date_Type`)

---

## Key Concepts

### Role-Playing Dimensions
Single dimension table used in multiple contexts via inactive relationships. Example: `Dim_Date_Reference` is reused as:
- `Dim_Date_FromDate` (assignment from date)
- `Dim_Date_CompletedOn` (completion date)
- `Dim_Date_FinalisedOn` (finalization date)

Activate with `USERELATIONSHIP()` in DAX measures.

### Bridge Tables (Many-to-Many)
Enable many-to-many relationships between dimensions. Examples:
- `Relative_Dates`: One date maps to multiple types ("Today" is also "This Month")
- `Assignment_Tags`: One assignment can have multiple tags
- `Team_Users`: Users can belong to multiple teams

### Date Key Pattern
Integer date format (YYYYMMDD) for efficient joins:
- November 18, 2025 → 20251118
- Faster than datetime comparisons
- Used across all date tables and fact tables

### Soft Deletes
Records marked as deleted with `IsDeleted = true` flag instead of physical deletion:
- Preserves audit trail
- Enables historical reporting
- Filtered out in Power Query

### Calculated Tables
DAX-generated tables computed at refresh:
- `Dim_Published_Worktemplates`: SUMMARIZE of published templates
- `Organisation_Info`: SUMMARIZE of tenant branding
- `Semantic_Model_*`: INFO.VIEW.* metadata tables

### Incremental Refresh
Large fact tables load only recent data with historical partitions:
- Reduces refresh time
- Lowers memory usage
- Retains compressed historical data

---

## Data Model Statistics

| Metric | Count |
|--------|-------|
| **Total Tables** | 105+ |
| **Fact Tables** | 12 |
| **Dimension Tables** | 40+ |
| **Bridge Tables** | 8 |
| **Parameter Tables** | 7 |
| **Boolean Helpers** | 5 |
| **RLS Tables** | 3 |
| **DAX Measures** | 200+ |
| **ERD Documents** | 8 |
| **Table Documents** | 105+ |
| **Total Relationships** | 150+ |

---

## Schema Design

### Star Schema Pattern
- **Central Facts**: `Assignments`, `Fact_Rosters`, `Fact_User_LogOn_Activities`
- **Surrounding Dimensions**: Date, User, Location, Status, Category
- **Benefits**: Fast aggregation, intuitive structure, optimal for Power BI

### Snowflake Pattern (Selective)
- **Hierarchical Dimensions**: `Dim_AssignmentPoints` (Area → Location → Point), `Dim_Fragments` (Groups → Sections → Fields)
- **Benefits**: Reduces redundancy, supports drill-down
- **Trade-off**: Additional joins (mitigated by calculated columns)

### Hybrid Approach
Combines star schema (performance) with snowflake hierarchies (flexibility) for optimal balance.

---

## Best Practices

### DAX Development
- ✅ Use `CALCULATE()` for context transition
- ✅ Use `USERELATIONSHIP()` for inactive relationships
- ✅ Store measures in `_Measures` table
- ✅ Prefer measures over calculated columns when possible
- ✅ Handle BLANK with `DIVIDE()` or `IF(ISBLANK())`

### Relationship Management
- ✅ Minimize bidirectional cross-filtering (use only for bridge tables)
- ✅ Document inactive relationships in ERD docs
- ✅ Avoid circular dependencies
- ✅ Use single-direction filtering (dimension → fact) by default

### Performance
- ✅ Use integer Date Keys (YYYYMMDD format)
- ✅ Consider aggregation tables for large facts (> 10M rows)
- ✅ Implement incremental refresh on large time-based facts
- ✅ Minimize calculated columns (prefer measures)
- ✅ Monitor query performance with Performance Analyzer

---

## Documentation Maintenance

**Last Updated:** November 19, 2025  
**Document Version:** 1.0  
**Semantic Model:** Obzervr - Data Cube

### Source of Truth
All documentation generated from TMDL metadata files:
```
Standard/Obzervr - Data Cube.SemanticModel/definition/
```

### Update Process
1. Update TMDL files when model changes
2. Regenerate ERD documentation using AI prompts (see README.md)
3. Update individual table documentation
4. Update this Home page with new table counts

---

## Additional Resources

- 📖 **[Full ERD Overview](ERD_Overview.md)** - Comprehensive technical documentation with architecture details
- 📖 **[Documentation README](README.md)** - Maintenance guide and AI regeneration prompts
- 🗂️ **[ERDs Folder](ERDs/)** - All 8 ERD documents
- 🗂️ **[Tables Folder](tables/)** - Individual table documentation (105+ files)

---

## Support & Feedback

For questions, issues, or feedback regarding this documentation:
- Review the specific ERD document for your area of interest
- Check individual table documentation for detailed column and relationship information
- Refer to DAX query pattern examples in each ERD document

---

**Happy Analyzing! 📊**
