# ERD #8: Parameters, Lookups & Helpers

## Overview

This ERD documents supporting tables that enable dynamic filtering, report parameterization, metadata inspection, and helper lookups across the Obzervr semantic model. These tables include DAX measures (_Measures), parameter tables for report controls, lookup tables for static data, boolean helper dimensions (Dim_Is_*), filtered dimension tables for tenant-specific selections, semantic model metadata tables, RLS (row-level security) support tables, and configuration tables.

**Key Components:**
- **DAX Measures Table**: _Measures table containing 200+ DAX measures organized by display folders
- **Parameter Tables**: Dynamic report controls for field selection, time period comparison, reading slicers (7 tables)
- **Lookup Tables**: Static reference data including shift duration, start times, aggregation types, field operators, label aliases (5 tables)
- **Boolean Helper Dimensions**: Dim_Is_* tables for filtering by assignment state (5 tables: Completed, Assigned, Outstanding, Resolved, AssignmentPoint_Active)
- **Filtered Dimension Tables**: Tenant-specific filtered lists for work templates, subsites, time series (6 tables)
- **Semantic Model Metadata**: Tables, columns, relationships, measures inspection tables (4 tables)
- **RLS Support Tables**: Row-level security role definitions and user-tenant mappings (3 tables)
- **Configuration Tables**: Organisation info, last refresh timestamp, version change history, enable flags (5 tables)

**Total Tables:** 35+ tables providing infrastructure and helper functionality across the semantic model

**Relationships:**
- Limited relationships (these tables are primarily standalone or use LOOKUPVALUE patterns)
- Filtered dimension tables relate to Dim_Tenant (Tenant_Id)
- Dim_Is_* tables have calculated columns that join to Assignments or Dim_AssignmentPoints
- Semantic_Model_Columns relates to Semantic_Model_Tables (Table to Name)
- Lkp_Label_Alias relates to Dim_Tenant (Tenant_Id)

## Table Inventory

### 1. _Measures (DAX Measures Table)
**Purpose:** Central table containing 200+ DAX measures organized into display folders for reporting and analytics. All measures for the semantic model are defined here rather than distributed across dimension or fact tables.

**Display Folder Categories:**
- **_Misc**: Miscellaneous measures (card tooltips, transparent color, report info, date range labels, primary/secondary colors, current local time)
- **Assignments**: Assignment counts, status metrics, completion rates
- **Field Measurements**: Field data aggregations, readings, calculations
- **Time Intelligence**: YTD, MTD, prior period comparisons, rolling averages
- **Users**: User activity metrics, audit counts
- **Templates**: Template usage, fragment statistics
- **Rosters**: Shift planning, attendance summaries
- **KPIs**: Key performance indicators across functional areas

**Sample Measures (from TMDL excerpt):**
```dax
// Misc - Card Tooltips
Misc - Card Tooltips = "  xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx    xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

// Misc - Transparent Colour
Misc - Transparent Colour = "#FFFFFF00"

// Misc - Report Info html
Misc - Report Info html = 
VAR TenantInfo = "This report offers the ability to monitor the usage of Obzervr application for tenant " & [Misc - Tenant Code List] & ". <br>" & "Row Level Security is applied according to user's analytic tenant permissions and the team assigned to.<br>"
VAR RLSInfo1 = "Current User (" & [Misc - Report User] & ")" & IF ( ISBLANK ( [Misc - Report User Analytic Permission] ), " has not been assigned to a Row Level Security role", SWITCH ( TRUE (), [Misc - Report User Analytic Permission] = 1, " has full analytic access to tenants " & [Misc - Analytic Tenant Code List], [Misc - Report User Analytic Permission] = 0, " only has access to assignments that assigned the user's team" ) ) & "."
VAR RLSInfo2 = "Current User (" & [Misc - Report User] & ")" & IF ( [Misc - RLS Role] = "Not Assigned", " has not been assigned to a Row Level Security role", SWITCH ( TRUE (), [Misc - RLS Role] = "Super User", " has full access to data", [Misc - RLS Role] = "Team User", " only has access to assignments that assigned the user's team" ) ) & "."
RETURN TenantInfo & IF ( [Misc - RLS Role] = "Activate RLS", RLSInfo1, RLSInfo2 )

// Misc - Date Range Label
Misc - Date Range Label = 
VAR DateMin = CALCULATE ( FORMAT ( MIN ( Dim_Date_FromDate[Date] ), "dd/MM/yyyy" ), ALL ( Dim_Tenant ) )
VAR Today = UTCNOW () + + TIME ( 0, MAX ( Dim_Tenant[Offset Minutes] ), 0 )
VAR DateMax = CALCULATE ( FORMAT ( IF ( Today < MAX ( Dim_Date_FromDate[Date] ), Today, MAX ( Dim_Date_FromDate[Date] ) ), "dd/MM/yyyy" ), ALL ( Dim_Tenant ) )
RETURN IF ( DateMin = DateMax, "The data displayed is for date " & FORMAT ( DateMin, "dd/MM/yyyy" ), "The data displayed is for the dates between " & FORMAT ( DateMin, "dd/MM/yyyy" ) & " and " & FORMAT ( DateMax, "dd/MM/yyyy" ) )

// Misc - Primary Colour
Misc - Primary Colour = MAX ( Dim_Tenant[Primary_Colour] )

// Misc - Secondary Colour
Misc - Secondary Colour = MAX ( Dim_Tenant[Secondary_Colour] )

// Misc - Current Local Time
Misc - Current Local Time = UTCNOW () + TIME ( 0, MAX ( Dim_Tenant[Offset Minutes] ), 0 )
```

**Pattern:** All measures in single table organized by displayFolder property. This centralizes measure management and simplifies semantic model maintenance.

**Lineage Tag:** 212311db-b240-45df-ab1a-3f25220ef377

---

### 2. Parameter_Asset_Hours_Type (Parameter Table)
**Purpose:** Parameter table for selecting between different asset hour measures (Actual Hours, Estimated Hours, Actual Duration, Estimated Duration, Actual Downtime, Estimated Downtime) in reports. Enables dynamic measure switching in visuals.

**Columns:** 3 (Parameter_Asset_Hours_Type [display name], Parameter_Asset_Hours_Type Fields [measure reference], Parameter_Asset_Hours_Type Order [sort order, hidden])

**Source (DAX Calculated Table):**
```dax
{
    ("Actual Hours", NAMEOF('_Measures'[Reading - Actual Hours]), 0),
    ("Estimated Hours", NAMEOF('_Measures'[Reading - Estimated Hours]), 1),
    ("Actual Duration", NAMEOF('_Measures'[Reading - Actual Duration]), 2),
    ("Estimated Duration", NAMEOF('_Measures'[Reading - Estimated Duration]), 3),
    ("Actual Downtime", NAMEOF('_Measures'[Reading - Actual Downtime]), 4),
    ("Estimated Downtime", NAMEOF('_Measures'[Reading - Estimated Downtime]), 5)
}
```

**Pattern:** Parameter table with three columns: display name (sorted by Order), Fields column (measure NAMEOF reference, marked as Parameter with kind=2, version=3), and hidden sort Order column.

**Relationships:** None (field selection logic uses selectedvalue in measures)

---

### 3. Parameter_Number_Measures (Parameter Table)
**Purpose:** Parameter table for selecting between integer measures in numeric reports. Enables users to switch between different numeric KPIs dynamically.

**Columns:** 3 (Parameter_Interger_Measures [display name], Parameter_Interger_Measures Fields [measure reference], Parameter_Interger_Measures Order [sort order, hidden])

**Source:** DAX calculated table (similar pattern to Parameter_Asset_Hours_Type)

**Pattern:** Field parameter pattern for dynamic measure selection

**Relationships:** None

---

### 4. Parameter_Previous_Period (Parameter Table)
**Purpose:** Parameter table for selecting previous period comparison options (e.g., Previous Month, Previous Quarter, Previous Year) in time intelligence calculations.

**Columns:** Display name and value columns for period selection

**Source:** DAX calculated table

**Pattern:** Slicer control table for time intelligence comparison measures

**Relationships:** None

---

### 5. Parameter_Reading_Slicer (Parameter Table)
**Purpose:** Parameter table controlling which field measurement readings are displayed in reports. Allows users to filter reading visualizations dynamically.

**Columns:** 2+ (Parameter_Reading_Slicer [display name], Parameter_Reading_Slicer Order [sort order, hidden])

**Source:** DAX calculated table

**Pattern:** Slicer control for field measurement filtering

**Relationships:** None

---

### 6. Parameter_Referenc_Date_Type (Parameter Table)
**Purpose:** Parameter table for selecting reference date type (FromDate vs CompletedOn vs FinalisedOn) for time-based analysis. Enables report users to switch date context dynamically.

**Columns:** Display name, value, and sort order

**Source:** DAX calculated table

**Pattern:** Date dimension role-playing parameter

**Relationships:** None (logic uses USERELATIONSHIP in measures based on parameter selection)

---

### 7. Parameter_Show_Cumulative (Parameter Table)
**Purpose:** Boolean parameter table for toggling cumulative vs period values in visualizations (e.g., cumulative assignments completed vs assignments completed in period).

**Columns:** Display name and boolean value

**Source:** DAX calculated table

**Pattern:** Toggle parameter (Yes/No or Show/Hide)

**Relationships:** None

---

### 8. Parameter_Show_Range (Parameter Table)
**Purpose:** Boolean parameter table for showing/hiding range indicators or error bars in visualizations.

**Columns:** Display name and boolean value

**Source:** DAX calculated table

**Pattern:** Toggle parameter for visual formatting

**Relationships:** None

---

### 9. Lkp_Shift_Duration (Lookup Table)
**Purpose:** Lookup table containing shift duration values from 0 to 24 hours. Provides hourly options for shift planning and filtering.

**Columns:** 1 (Value [int64, 0 to 24])

**Source (DAX Calculated Table):**
```dax
GENERATESERIES(0, 24, 1)
```

**Pattern:** Simple integer series for shift hour selection

**Relationships:** None (used via slicers and filtering)

---

### 10. Lkp_Shift_Start_Time (Lookup Table)
**Purpose:** Lookup table containing shift start time options (likely 0-23 representing hours of the day). Provides hourly start time options for shift configuration.

**Columns:** 1 (Value [int64])

**Source:** DAX calculated table (GENERATESERIES pattern inferred)

**Pattern:** Integer series for shift start hour selection

**Relationships:** None

---

### 11. Lkp_Aggregation_Type (Lookup Table)
**Purpose:** Lookup table defining aggregation type options (Sum, Average, etc.) for field measurement aggregation in TSFM reports. Provides consistent aggregation method selection.

**Columns:** 2 (Aggregation_Type [string, sorted by Sort_Order], Sort_Order [int64])

**Source (Power Query M):**
```m
let
    Source = Table.FromRows(Json.Document(Binary.Decompress(Binary.FromText("...", BinaryEncoding.Base64), Compression.Deflate)), ...),
    #"Changed Type" = Table.TransformColumnTypes(Source,{{"Aggregation_Type", type text}, {"Sort_Order", Int64.Type}})
in
    #"Changed Type"
```

**Data (inferred):** Sum (0), Average (1), Min (2), Max (3), Count (4)

**Pattern:** Static lookup with sort order

**Relationships:** None

---

### 12. Lkp_Assignment_FieldOperators (Lookup Table, Hidden)
**Purpose:** Calculated lookup table containing comma-separated lists of field operators for each assignment. Provides quick access to operator names without traversing relationships.

**Columns:** 2 (Assignment_Id [string, hidden], Field_Operators [string, hidden])

**Source (DAX Calculated Table):**
```dax
SUMMARIZECOLUMNS(
    'Assignments'[Assignment_Id],
    "Field_Operators", '_Measures'[ASGT - Field Operators]
)
```

**Pattern:** Denormalized lookup created from measure evaluation for performance

**Relationships:** None (hidden helper table)

---

### 13. Lkp_Label_Alias (Lookup Table)
**Purpose:** Label localization and aliasing table. Provides tenant-specific label translations or aliases for field names, status labels, and UI elements. Supports multi-language or tenant-specific terminology.

**Columns:** 6 (Tenant_Id, Id, Label_Key, Label_Value, Last_Updated, Created_Date)

**Source:** SQL query from LabelAlias table with tenant filtering and soft delete filtering (IsDeleted = false)

**Pattern:** Tenant-specific localization table

**Relationships:**
- Many-to-one to Dim_Tenant (Tenant_Id) - Relationship ID: (inferred from grep search)

---

### 14. Dim_Is_Completed (Boolean Helper Dimension)
**Purpose:** Lookup table defining completion statuses (Completed or Not Completed) used for filtering assignments and KPI logic. Two-row dimension for boolean filtering.

**Columns:** 2 (Name [string, display name], Value [string, underlying value])

**Source (Power Query M):**
```m
let
    Source = Table.FromRows(Json.Document(Binary.Decompress(Binary.FromText("i45Wcs7PLchJLUlNUdJRMlSK1YlW8sxLhooBhQyUYmMB", BinaryEncoding.Base64), Compression.Deflate)), ...),
    #"Changed Type" = Table.TransformColumnTypes(Source,{{"Name", type text}, {"Value", type text}})
in
    #"Changed Type"
```

**Data:** Two rows - "Completed" and "Not Completed" (inferred)

**Pattern:** Boolean helper dimension with Name/Value structure

**Relationships:**
- Assignments[Is_Completed_Text] to Dim_Is_Completed[Value] (calculated column pattern, relationship ID: inferred)

---

### 15. Dim_Is_Assigned (Boolean Helper Dimension)
**Purpose:** Lookup table defining assignment statuses (Assigned or Not Assigned) used for filtering and labeling in reports. Enables filtering by whether assignments have been assigned to a user.

**Columns:** 2 (Name [string, display name], Value [string, underlying value])

**Source (Power Query M):**
```m
let
    Source = Table.FromRows(Json.Document(Binary.Decompress(Binary.FromText("i45WciwuzkzPS01R0lEyVIrViVYKzUtECBkoxcYCAA==", BinaryEncoding.Base64), Compression.Deflate)), ...),
    #"Changed Type" = Table.TransformColumnTypes(Source,{{"Name", type text}, {"Value", type text}})
in
    #"Changed Type"
```

**Data:** Two rows - "Assigned" and "Not Assigned"

**Pattern:** Boolean helper dimension

**Relationships:**
- Assignments[Is_Assigned_Text] to Dim_Is_Assigned[Value] (calculated column pattern, relationship ID: inferred)

---

### 16. Dim_Is_Outstanding (Boolean Helper Dimension)
**Purpose:** Lookup table defining outstanding statuses (Outstanding or Not Outstanding) for assignment filtering. Identifies assignments requiring attention or action.

**Columns:** 2 (Name, Value)

**Source:** Power Query M (binary compressed JSON pattern)

**Data:** Two rows - "Outstanding" and "Not Outstanding"

**Pattern:** Boolean helper dimension

**Relationships:**
- Assignments[Is_Outstanding_Text] to Dim_Is_Outstanding[Value] (calculated column pattern, relationship ID: inferred)

---

### 17. Dim_Is_Resolved (Boolean Helper Dimension)
**Purpose:** Lookup table defining resolution statuses (Resolved or Not Resolved) for exception and issue tracking. Identifies whether exceptions or issues have been addressed.

**Columns:** 2 (Name, Value)

**Source:** Power Query M

**Data:** Two rows - "Resolved" and "Not Resolved"

**Pattern:** Boolean helper dimension

**Relationships:**
- Assignment_FieldMeasurement_Exceptions[Is_Resolved_Text] to Dim_Is_Resolved[Value] (calculated column pattern, relationship ID: inferred)

---

### 18. Dim_Is_AssignmentPoint_Active (Boolean Helper Dimension)
**Purpose:** Lookup table defining active statuses (Active or Inactive) for assignment points. Filters assignment points by their active status for current vs historical reporting.

**Columns:** 2 (Name, Value)

**Source:** Power Query M

**Data:** Two rows - "Active" and "Inactive"

**Pattern:** Boolean helper dimension

**Relationships:**
- Dim_AssignmentPoints[Is_Active_Text] to Dim_Is_AssignmentPoint_Active[Value] (calculated column pattern, relationship ID: inferred)

---

### 19. WorkTemplate_Filtered_1 (Filtered Dimension)
**Purpose:** Filtered list of work templates (Template List 1) available for each tenant. Limits template selections in reports to tenant-specific templates defined in Dim_Tenant[Template_List_1].

**Columns:** 2 (Tenant_Id, WorkTemplate_Name)

**Source (Power Query M):**
```m
let
    Source = Dim_Tenant,
    #"Removed Other Columns" = Table.SelectColumns(Source,{"Tenant_Id", "Template_List_1"}),
    #"Split Column by Delimiter" = Table.ExpandListColumn(Table.TransformColumns(#"Removed Other Columns", {{"Template_List_1", Splitter.SplitTextByDelimiter(",", QuoteStyle.Csv), ...}}), "Template_List_1"),
    #"Changed Type" = Table.TransformColumnTypes(#"Split Column by Delimiter",{{"Template_List_1", type text}}),
    #"Renamed Columns" = Table.RenameColumns(#"Changed Type",{{"Template_List_1", "WorkTemplate_Name"}})
in
    #"Renamed Columns"
```

**Pattern:** Comma-delimited string split into rows for tenant-specific filtering

**Relationships:**
- WorkTemplate_Filtered_1[Tenant_Id] to Dim_Tenant[Tenant_Id] - Relationship ID: (inferred)

---

### 20. WorkTemplate_Filtered_2 (Filtered Dimension)
**Purpose:** Filtered list of work templates (Template List 2) for each tenant. Second template filtering dimension sourced from Dim_Tenant[Template_List_2].

**Columns:** 2 (Tenant_Id, WorkTemplate_Name)

**Source:** Power Query M (similar to WorkTemplate_Filtered_1 but from Template_List_2 column)

**Pattern:** Comma-delimited string split into rows

**Relationships:**
- WorkTemplate_Filtered_2[Tenant_Id] to Dim_Tenant[Tenant_Id]

---

### 21. Subsite_Filtered_1 (Filtered Dimension)
**Purpose:** Filtered list of subsites (Subsite List 1) available for each tenant. Limits subsite selections to tenant-specific subsites defined in Dim_Tenant[Subsite_List_1].

**Columns:** 2 (Tenant_Id, Subsite_Name or Subsite_Id)

**Source:** Power Query M (comma-delimited split from Dim_Tenant)

**Pattern:** Tenant-specific filtering dimension

**Relationships:**
- Subsite_Filtered_1[Tenant_Id] to Dim_Tenant[Tenant_Id]

---

### 22. Subsite_Filtered_2 (Filtered Dimension)
**Purpose:** Filtered list of subsites (Subsite List 2) for each tenant. Second subsite filtering dimension.

**Columns:** 2 (Tenant_Id, Subsite_Name or Subsite_Id)

**Source:** Power Query M

**Pattern:** Tenant-specific filtering dimension

**Relationships:**
- Subsite_Filtered_2[Tenant_Id] to Dim_Tenant[Tenant_Id]

---

### 23. TimeSeries_Filtered_1 (Filtered Dimension)
**Purpose:** Filtered list of time series / groups (Series List 1) available for each tenant. Limits time series selections to tenant-specific series defined in Dim_Tenant.

**Columns:** 2+ (Tenant_Id, Series_Name or Series_Identifier)

**Source:** Power Query M (comma-delimited split from Dim_Tenant)

**Pattern:** Tenant-specific filtering dimension for time series

**Relationships:**
- TimeSeries_Filtered_1[Tenant_Id] to Dim_Tenant[Tenant_Id]
- TimeSeries[Series_Name] to TimeSeries_Filtered_1[Series_Name] (inactive or filtered relationship)

---

### 24. TimeSeries_Filtered_2 (Filtered Dimension)
**Purpose:** Filtered list of time series / groups (Series List 2) for each tenant. Second time series filtering dimension.

**Columns:** 2+ (Tenant_Id, Series_Identifier or Series_Name)

**Source:** Power Query M

**Pattern:** Tenant-specific filtering dimension

**Relationships:**
- TimeSeries_Filtered_2[Tenant_Id] to Dim_Tenant[Tenant_Id]
- TimeSeries[Series_Identifier] to TimeSeries_Filtered_2[Series_Identifier] (inactive or filtered relationship)

---

### 25. Semantic_Model_Tables (Metadata Table)
**Purpose:** Metadata table listing all tables in the semantic model with properties (ID, Name, Model, DataCategory, Description, IsHidden, StorageMode, TableStorage, Expression, ShowAsVariationOnly, IsPrivate, CalculationGroupPrecedence, LineageTag). Used for model documentation and analysis.

**Columns:** 13 (ID, Name, Model, DataCategory, Description, IsHidden, StorageMode, TableStorage, Expression, ShowAsVariationOnly, IsPrivate, CalculationGroupPrecedence, LineageTag)

**Source (DAX Calculated Table):**
```dax
INFO.VIEW.TABLES()
```

**Pattern:** Dynamic metadata inspection using INFO.VIEW.TABLES() DMV function

**Relationships:** None (metadata reference table)

---

### 26. Semantic_Model_Columns (Metadata Table)
**Purpose:** Metadata table listing all columns across all tables with properties (ID, Name, Table, DataType, DataCategory, Description, IsHidden, IsUnique, IsKey, IsNullable, Alignment, SummarizeBy, ColumnStorage, Type, SourceColumn, Expression, FormatString, IsAvailableInMDX, SortByColumn, GroupingBehavior, SourceProviderType, DisplayFolder, AlternateOf, LineageTag). Used for model documentation and lineage tracking.

**Columns:** 24 (ID, Name, Table, DataType, DataCategory, Description, IsHidden, IsUnique, IsKey, IsNullable, Alignment, SummarizeBy, ColumnStorage, Type, SourceColumn, Expression, FormatString, IsAvailableInMDX, SortByColumn, GroupingBehavior, SourceProviderType, DisplayFolder, AlternateOf, LineageTag)

**Source (DAX Calculated Table):**
```dax
INFO.VIEW.COLUMNS()
```

**Pattern:** Dynamic metadata inspection using INFO.VIEW.COLUMNS() DMV function

**Relationships:**
- Semantic_Model_Columns[Table] to Semantic_Model_Tables[Name] - Relationship ID: (inferred)

---

### 27. Semantic_Model_Relationships (Metadata Table)
**Purpose:** Metadata table listing all relationships in the semantic model with properties (ID, Name, Relationship [formatted], Model, IsActive, CrossFilteringBehavior, RelyOnReferentialIntegrity, FromTable, FromColumn, FromCardinality, ToTable, ToColumn, ToCardinality, State, SecurityFilteringBehavior). Includes calculated Description column with human-readable relationship explanation. Used for model documentation and relationship analysis.

**Columns:** 15+ (ID, Name, Relationship, Model, IsActive, CrossFilteringBehavior, RelyOnReferentialIntegrity, FromTable, FromColumn, FromCardinality, ToTable, ToColumn, ToCardinality, State, SecurityFilteringBehavior, Description [calculated], SeqNumber [calculated])

**Calculated Column - Description:**
```dax
Description = 
"The relationship links '" &
[FromTable] & "'[" & [FromColumn] & "] to '" &
[ToTable] & "'[" & [ToColumn] & "]. It is a " &
SWITCH(
    TRUE(),
    [FromCardinality] = "One" && [ToCardinality] = "One", "one-to-one relationship (1:1)",
    [FromCardinality] = "One" && [ToCardinality] = "Many", "one-to-many relationship (1:*)",
    [FromCardinality] = "Many" && [ToCardinality] = "One", "many-to-one relationship (Many:1)",
    [FromCardinality] = "Many" && [ToCardinality] = "Many", "many-to-many relationship (*:*)",
    "relationship"
) &
" and supports " &
SWITCH(
    [CrossFilteringBehavior],
    "BothDirections", "bidirectional cross-filtering, meaning that filters applied in either table will affect the other",
    "Single", "single-direction cross-filtering, meaning that filters flow only from the 'from' table to the 'to' table",
    [CrossFilteringBehavior]
) & "."
```

**Source (DAX Calculated Table):**
```dax
INFO.VIEW.RELATIONSHIPS()
```

**Pattern:** Dynamic metadata inspection with human-readable description generation

**Relationships:** None (metadata reference table)

---

### 28. Semantic_Model_Measures (Metadata Table)
**Purpose:** Metadata table listing all measures in the semantic model with properties (ID, Name, Table, Description, DataType, Expression, FormatString, IsHidden, State, KPIID, IsSimpleMeasure, DisplayFolder, DetailRowsDefinition, DataCategory, FormatStringDefinition, LineageTag). Used for measure inventory and DAX code documentation.

**Columns:** 16 (ID, Name, Table, Description, DataType, Expression, FormatString, IsHidden, State, KPIID, IsSimpleMeasure, DisplayFolder, DetailRowsDefinition, DataCategory, FormatStringDefinition, LineageTag)

**Source (DAX Calculated Table):**
```dax
INFO.VIEW.MEASURES()
```

**Pattern:** Dynamic metadata inspection for measures

**Relationships:** None (metadata reference table)

---

### 29. RLS_Roles (RLS Support Table)
**Purpose:** RLS (row-level security) role definitions. Contains numeric id and role name used for mapping users to RLS roles. Defines available security roles (e.g., Super User, Team User).

**Columns:** 2 (Id [int64, sorted], Name [string, sorted by Id])

**Source (Power Query M):**
```m
let
    Source = Table.FromRows(Json.Document(Binary.Decompress(Binary.FromText("i45WMlDSUXJMLsksSyxJVQjyCVaK1YlWMgQKhqQm5iqEFqcWgUWMgCLBpQWpRVChWAA=", BinaryEncoding.Base64), Compression.Deflate)), ...),
    #"Changed Type" = Table.TransformColumnTypes(Source,{{"Id", Int64.Type}, {"Name", type text}})
in
    #"Changed Type"
```

**Data (inferred):** 0 = Not Assigned, 1 = Super User, 2 = Team User, 3 = Activate RLS

**Pattern:** Static role definition table for RLS

**Relationships:** None (reference table for RLS logic)

**Query Group:** RLS

---

### 30. RLS_Users (RLS Support Table)
**Purpose:** RLS user mapping table. Maps Power BI users (USERPRINCIPALNAME) to RLS roles. Used in RLS DAX filter expressions to determine user permissions.

**Columns:** 2+ (User_Email or User_Principal_Name, Role_Id or Role_Name)

**Source:** Power Query M (likely SQL source or manual entry)

**Pattern:** User to role mapping for RLS implementation

**Relationships:** None (used in RLS filter expressions via USERPRINCIPALNAME() function)

**Query Group:** RLS

---

### 31. RLS_Tenant_User_Analytic (RLS Support Table)
**Purpose:** RLS tenant-user analytic permission mapping. Defines which users have analytic access to which tenants. Used for multi-tenant RLS filtering.

**Columns:** 3+ (User_Email or User_Principal_Name, Tenant_Id, Analytic_Permission [boolean or numeric])

**Source:** Power Query M (SQL query inferred)

**Pattern:** User-tenant permission matrix for RLS

**Relationships:** None (used in RLS filter expressions)

**Query Group:** RLS

---

### 32. Last_Refreshed_UTC (Configuration Table)
**Purpose:** Provides the exact UTC timestamp indicating when the Power BI dataset was last refreshed. Single-row table displaying data currency for monitoring refresh schedules and data freshness.

**Columns:** 1 (DateTime [datetime, formatted as dd/MM/yyyy HH:mm])

**Source (Power Query M):**
```m
let
    Source = DateTimeZone.FixedUtcNow(),
    #"Converted to Table" = #table(1, {{Source}}),
    #"Renamed Columns" = Table.RenameColumns(#"Converted to Table",{{"Column1", "DateTime"}}),
    #"Changed Type" = Table.TransformColumnTypes(#"Renamed Columns",{{"DateTime", type datetimezone}})
in
    #"Changed Type"
```

**Pattern:** Single-row configuration table with refresh timestamp

**Relationships:** None

**Query Group:** Data & Time

---

### 33. Organisation_Info (Configuration Table, Calculated)
**Purpose:** Contains basic organisation information such as the organisation logo (base64 encoded) and unique organisation key. Used to display branding and identify the organisation within reports and dashboards. Denormalized from Dim_Tenant for easier access.

**Columns:** 2 (Logo_Base64 [string, DataCategory: ImageUrl], OrganisationKey [string])

**Source (DAX Calculated Table):**
```dax
SUMMARIZE(
    Dim_Tenant,
    Dim_Tenant[Logo_Base64],
    Dim_Tenant[OrganisationKey]
)
```

**Pattern:** Calculated summary table for report branding

**Relationships:** None (denormalized reference table)

---

### 34. Version_Change_History (Configuration Table, Hidden)
**Purpose:** Version change history tracking table. Records semantic model version numbers, change details, dates, and sort order. Hidden from users but available for documentation and version tracking. Provides audit trail of model evolution.

**Columns:** 4 (Version [string, sorted by Sort Order, hidden], Details [string, hidden], Date [date, hidden], Sort Order [int64, hidden])

**Source (Power Query M):**
```m
let
    Source = Table.FromRows(Json.Document(Binary.Decompress(Binary.FromText("bZNPb+MgEMW/yijnao0d59/RbRKpUrNbJemp2wM1JEGLwQJcbb79DsRQt17JF0v83sy8efP6OvnIf5DJ3eRRCSeohLWhJ4f/pMjIPCtIUeJPPnm7Sy8fDKeOM7inlsOOU9sZbj0xHRDF/4g9b7Vx0NLzDZgNgOkX4ELVmQOVEhx9l9z+7ggpFsCoo2B1Z2oOTkOlqLw6UcP63pfMB3JllMvxp2IMqz+q2vCGK4dT7vkJm774JhYZfj01G1OVteKsPDacdUoysozUPFJFop60/tO1cAzN+97Qm1V8vxi/f2lZcGgruGSxUCj6kzbcC5QDgeVY4EHLrlHWu/JiuQEmkLZCq97A7xKrKDFNEoeLODkITs9hr61DmQSjszlJcSCRLhO9UU6465eJB0AK0CwB0U04aQPPkuKscZ+BJ1meJz7Faf5pMf/gcgV1P7hXGezqWQvlQqsZJhllfOF8OpbZKN/x8bDdfdIBxpga9B5duHWD8Y4y5VgGTQ+UX9cvJa+VHYtgYkiZRFLWFinwzK/PNUz6aRrq+y/ijQQmJW35bWtHXDdwhSp1iI0nl32wA7kYk0/0nUuopKA2WNwnOjxPCVul50fetBJDCltDz8HjzV9naO1g7a9ypxmqUcXghMeOi72Zf9th0FyNNQ+8wb3jAd9opusuHKjD4AajB3yBoXv7Bw==", BinaryEncoding.Base64), Compression.Deflate)), ...),
    #"Changed Type" = Table.TransformColumnTypes(Source,{{"Version", type text}, {"Details", type text}, {"Date", type date}, {"Sort Order", Int64.Type}})
in
    #"Changed Type"
```

**Pattern:** Static version history table with binary compressed JSON source

**Relationships:** None

**IsHidden:** True

---

### 35. EnableTSFMAssignmentPoint (Enable Flag)
**Purpose:** Boolean parameter or single-row table controlling whether TSFM (Time Series Field Measurement) assignment point features are enabled. Used to conditionally load data or show/hide features.

**Columns:** 1 (Enable [boolean] or Value [string/int])

**Source:** Power Query parameter or calculated table

**Pattern:** Feature flag table

**Relationships:** None

---

### 36. UsePointNameOnlyAsPoint (Enable Flag)
**Purpose:** Boolean parameter controlling whether to use point name only (without full hierarchy) for assignment point display. Simplifies point naming in certain report contexts.

**Columns:** 1 (Value [boolean] or similar)

**Source:** Power Query parameter

**Pattern:** Feature flag table

**Relationships:** None

---

## Key Data Model Patterns

### 1. Parameter Tables (Field Selection Pattern)
Seven parameter tables (Parameter_Asset_Hours_Type, Parameter_Number_Measures, Parameter_Previous_Period, Parameter_Reading_Slicer, Parameter_Referenc_Date_Type, Parameter_Show_Cumulative, Parameter_Show_Range) implement the field parameter pattern for dynamic measure and dimension selection in reports.

**Implementation:**
```dax
// Parameter_Asset_Hours_Type source
{
    ("Actual Hours", NAMEOF('_Measures'[Reading - Actual Hours]), 0),
    ("Estimated Hours", NAMEOF('_Measures'[Reading - Estimated Hours]), 1),
    // ... more options
}
```

**Structure:**
- Column 1: Display name (shown in slicer, sorted by Order column)
- Column 2: Fields column with NAMEOF() measure reference (marked with ParameterMetadata extendedProperty kind=2, version=3)
- Column 3: Hidden sort order column

**Usage in Measures:**
```dax
Selected Asset Hours = 
VAR SelectedField = SELECTEDVALUE(Parameter_Asset_Hours_Type[Parameter_Asset_Hours_Type Fields])
RETURN
SWITCH(SelectedField,
    NAMEOF('_Measures'[Reading - Actual Hours]), [Reading - Actual Hours],
    NAMEOF('_Measures'[Reading - Estimated Hours]), [Reading - Estimated Hours],
    // ... other cases
    BLANK()
)
```

**Benefits:**
- Dynamic field selection without multiple report pages
- User-driven measure switching in visuals
- Simplified report maintenance

### 2. Boolean Helper Dimensions (Dim_Is_* Pattern)
Five boolean helper dimensions (Dim_Is_Completed, Dim_Is_Assigned, Dim_Is_Outstanding, Dim_Is_Resolved, Dim_Is_AssignmentPoint_Active) implement consistent two-row lookup pattern for filtering.

**Structure:**
- Two rows per table (e.g., "Completed" / "Not Completed")
- Name column: Display-friendly text for report labels
- Value column: Underlying value (often matches calculation logic)
- Source: Power Query M with binary compressed JSON

**Pattern:**
```m
let
    Source = Table.FromRows(Json.Document(Binary.Decompress(Binary.FromText("...", BinaryEncoding.Base64), Compression.Deflate)), ...),
    #"Changed Type" = Table.TransformColumnTypes(Source,{{"Name", type text}, {"Value", type text}})
in
    #"Changed Type"
```

**Usage:**
- Fact/dimension tables have calculated columns (e.g., Assignments[Is_Completed_Text]) that map to Dim_Is_Completed[Value]
- Relationships enable slicer filtering
- Consistent boolean filtering UI across reports

**Benefits:**
- User-friendly labels (vs "True"/"False" or 1/0)
- Consistent filtering pattern
- Easy localization (modify Name column)

### 3. Filtered Dimension Tables (Tenant-Specific Lists)
Six filtered dimension tables (WorkTemplate_Filtered_1, WorkTemplate_Filtered_2, Subsite_Filtered_1, Subsite_Filtered_2, TimeSeries_Filtered_1, TimeSeries_Filtered_2) split comma-delimited lists from Dim_Tenant into rows for tenant-specific filtering.

**Pattern:**
```m
let
    Source = Dim_Tenant,
    #"Removed Other Columns" = Table.SelectColumns(Source,{"Tenant_Id", "Template_List_1"}),
    #"Split Column by Delimiter" = Table.ExpandListColumn(Table.TransformColumns(#"Removed Other Columns", {{"Template_List_1", Splitter.SplitTextByDelimiter(",", QuoteStyle.Csv), ...}}), "Template_List_1"),
    #"Changed Type" = Table.TransformColumnTypes(#"Split Column by Delimiter",{{"Template_List_1", type text}}),
    #"Renamed Columns" = Table.RenameColumns(#"Changed Type",{{"Template_List_1", "WorkTemplate_Name"}})
in
    #"Renamed Columns"
```

**Result:**
- Input: Dim_Tenant[Template_List_1] = "Template A, Template B, Template C"
- Output: Three rows in WorkTemplate_Filtered_1: (TenantId, "Template A"), (TenantId, "Template B"), (TenantId, "Template C")

**Usage:**
- Slicer tables that automatically filter to tenant's allowed values
- Two versions (_1, _2) allow different filtering contexts or backup lists

**Benefits:**
- Tenant isolation without multiple semantic models
- Dynamic list management via Dim_Tenant configuration
- Avoids hard-coded filter lists

### 4. Semantic Model Metadata Tables (INFO.VIEW.* Functions)
Four metadata tables (Semantic_Model_Tables, Semantic_Model_Columns, Semantic_Model_Relationships, Semantic_Model_Measures) provide dynamic introspection of the semantic model structure using DAX DMV functions.

**Implementation:**
```dax
// Semantic_Model_Tables
INFO.VIEW.TABLES()

// Semantic_Model_Columns
INFO.VIEW.COLUMNS()

// Semantic_Model_Relationships
INFO.VIEW.RELATIONSHIPS()

// Semantic_Model_Measures
INFO.VIEW.MEASURES()
```

**Calculated Column Example (Semantic_Model_Relationships):**
```dax
Description = 
"The relationship links '" & [FromTable] & "'[" & [FromColumn] & "] to '" & [ToTable] & "'[" & [ToColumn] & "]. It is a " &
SWITCH(TRUE(), [FromCardinality] = "One" && [ToCardinality] = "One", "one-to-one relationship (1:1)", ...) &
" and supports " & SWITCH([CrossFilteringBehavior], "BothDirections", "bidirectional cross-filtering...", ...) & "."
```

**Benefits:**
- Self-documenting semantic model
- Dynamic inventory for governance
- Lineage tracking via LineageTag columns
- Human-readable relationship descriptions
- Useful for model analysis reports

### 5. Centralized Measures Table (_Measures)
All 200+ DAX measures defined in single _Measures table rather than distributed across dimension/fact tables. Organized via displayFolder property.

**Structure:**
- Table type: No source columns (pure measures container)
- Measures organized by displayFolder: _Misc, Assignments, Field Measurements, Time Intelligence, Users, Templates, Rosters, KPIs
- No data rows (table exists solely for measure organization)

**Benefits:**
- Centralized measure management
- Easier measure search and maintenance
- Consistent measure naming convention
- Display folders create logical grouping in Field List
- Simplifies semantic model deployment and versioning

### 6. RLS Support Tables (Row-Level Security)
Three RLS tables (RLS_Roles, RLS_Users, RLS_Tenant_User_Analytic) support multi-tenant row-level security implementation.

**Pattern:**
- RLS_Roles: Static role definitions (Super User, Team User, Not Assigned, Activate RLS)
- RLS_Users: User email → role mapping
- RLS_Tenant_User_Analytic: User email → tenant → analytic permission mapping

**Usage in RLS Expressions:**
```dax
// Example RLS filter on Dim_Tenant
VAR CurrentUser = USERPRINCIPALNAME()
VAR UserRole = LOOKUPVALUE(RLS_Users[Role_Name], RLS_Users[User_Email], CurrentUser)
VAR UserTenants = FILTER(RLS_Tenant_User_Analytic, RLS_Tenant_User_Analytic[User_Email] = CurrentUser)
RETURN
SWITCH(UserRole,
    "Super User", TRUE(),
    "Team User", Dim_Tenant[Tenant_Id] IN VALUES(UserTenants[Tenant_Id]),
    FALSE()
)
```

**Benefits:**
- Dynamic user permission management
- Multi-tenant data isolation
- Role-based access control
- No hard-coded user lists in RLS expressions

### 7. Configuration Tables (Single-Row Reference)
Three single-row or near-static tables (Last_Refreshed_UTC, Organisation_Info, Version_Change_History) provide configuration and metadata.

**Last_Refreshed_UTC Pattern:**
```m
let
    Source = DateTimeZone.FixedUtcNow(),  // Evaluated at refresh time
    #"Converted to Table" = #table(1, {{Source}}),
    #"Renamed Columns" = Table.RenameColumns(#"Converted to Table",{{"Column1", "DateTime"}}),
    #"Changed Type" = Table.TransformColumnTypes(#"Renamed Columns",{{"DateTime", type datetimezone}})
in
    #"Changed Type"
```

**Result:** Single row with current UTC timestamp captured at refresh

**Organisation_Info Pattern (Calculated Table):**
```dax
SUMMARIZE(
    Dim_Tenant,
    Dim_Tenant[Logo_Base64],
    Dim_Tenant[OrganisationKey]
)
```

**Result:** Denormalized organisation branding data for report consumption

**Benefits:**
- Data freshness indicator (Last_Refreshed_UTC)
- Simplified report branding (Organisation_Info)
- Semantic model change tracking (Version_Change_History)

---

## Common DAX Query Patterns

### Parameter Table Usage

**Get selected parameter value:**
```dax
EVALUATE
ADDCOLUMNS(
    {1},
    "Selected Asset Hours Type", SELECTEDVALUE(Parameter_Asset_Hours_Type[Parameter_Asset_Hours_Type]),
    "Selected Field", SELECTEDVALUE(Parameter_Asset_Hours_Type[Parameter_Asset_Hours_Type Fields]),
    "Selected Previous Period", SELECTEDVALUE(Parameter_Previous_Period[Period]),
    "Show Cumulative", SELECTEDVALUE(Parameter_Show_Cumulative[Value])
)
```

**Dynamic measure with parameter:**
```dax
Dynamic Asset Hours = 
VAR SelectedField = SELECTEDVALUE(Parameter_Asset_Hours_Type[Parameter_Asset_Hours_Type Fields])
RETURN
SWITCH(
    SelectedField,
    NAMEOF('_Measures'[Reading - Actual Hours]), [Reading - Actual Hours],
    NAMEOF('_Measures'[Reading - Estimated Hours]), [Reading - Estimated Hours],
    NAMEOF('_Measures'[Reading - Actual Duration]), [Reading - Actual Duration],
    [Reading - Actual Hours]  // Default
)
```

### Boolean Helper Dimension Queries

**Filter by completion status:**
```dax
EVALUATE
CALCULATETABLE(
    SUMMARIZE(
        Assignments,
        Assignments[Assignment_Number],
        Dim_Date_CompletedOn[Date]
    ),
    Dim_Is_Completed[Name] = "Completed"
)
ORDER BY Dim_Date_CompletedOn[Date] DESC
```

**Count assignments by assigned status:**
```dax
EVALUATE
SUMMARIZE(
    Assignments,
    Dim_Is_Assigned[Name],
    "Count", COUNTROWS(Assignments)
)
```

### Semantic Model Metadata Queries

**List all tables with row counts:**
```dax
EVALUATE
ADDCOLUMNS(
    Semantic_Model_Tables,
    "Row Count", CALCULATE(COUNTROWS(RELATEDTABLE([Name])))
)
ORDER BY [Row Count] DESC
```

**List all measures in display folder:**
```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Semantic_Model_Measures,
        "Measure", [Name],
        "Table", [Table],
        "Display Folder", [DisplayFolder],
        "Description", [Description],
        "DAX Expression", [Expression]
    ),
    [Display Folder] = "Assignments"
)
ORDER BY [Measure]
```

**List all relationships for a table:**
```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Semantic_Model_Relationships,
        "Relationship", [Relationship],
        "From", [FromTable] & "[" & [FromColumn] & "]",
        "To", [ToTable] & "[" & [ToColumn] & "]",
        "Description", [Description],
        "Active", [IsActive],
        "Cross-Filtering", [CrossFilteringBehavior]
    ),
    [FromTable] = "Assignments" || [ToTable] = "Assignments"
)
```

**Find calculated columns:**
```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Semantic_Model_Columns,
        "Table", [Table],
        "Column", [Name],
        "Type", [Type],
        "DAX Expression", [Expression]
    ),
    [Type] = "Calculated" && NOT ISBLANK([DAX Expression])
)
ORDER BY [Table], [Column]
```

### RLS Support Table Queries

**Check user permissions:**
```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        RLS_Users,
        "User", [User_Email],
        "Role", [Role_Name]
    ),
    [User_Email] = "user@example.com"
)
```

**List tenant access per user:**
```dax
EVALUATE
SELECTCOLUMNS(
    RLS_Tenant_User_Analytic,
    "User", [User_Email],
    "Tenant", RELATED(Dim_Tenant[Tenant_Code]),
    "Analytic Permission", [Analytic_Permission]
)
ORDER BY [User], [Tenant]
```

### Configuration Table Queries

**Check data freshness:**
```dax
EVALUATE
ADDCOLUMNS(
    Last_Refreshed_UTC,
    "Hours Since Refresh", DIVIDE(
        DATEDIFF(Last_Refreshed_UTC[DateTime], NOW(), MINUTE),
        60,
        2
    )
)
```

**Get organisation branding:**
```dax
EVALUATE
SELECTCOLUMNS(
    Organisation_Info,
    "Organisation Key", [OrganisationKey],
    "Logo", [Logo_Base64]
)
```

### Filtered Dimension Queries

**List allowed templates for tenant:**
```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        WorkTemplate_Filtered_1,
        "Tenant", RELATED(Dim_Tenant[Tenant_Code]),
        "Template", [WorkTemplate_Name]
    ),
    RELATED(Dim_Tenant[Tenant_Code]) = "ACME"
)
ORDER BY [Template]
```

**Compare Template_List_1 vs Template_List_2:**
```dax
DEFINE
    VAR List1 = SELECTCOLUMNS(WorkTemplate_Filtered_1, "Tenant", [Tenant_Id], "Template", [WorkTemplate_Name], "List", "List 1")
    VAR List2 = SELECTCOLUMNS(WorkTemplate_Filtered_2, "Tenant", [Tenant_Id], "Template", [WorkTemplate_Name], "List", "List 2")
EVALUATE
UNION(List1, List2)
ORDER BY [Tenant], [List], [Template]
```

---

## Related Documentation

### ERD Cross-References
- **ERD #1: Assignment Core Model** - Assignments (relationships to Dim_Is_* tables via calculated columns)
- **ERD #2: Date Dimensions & Time Intelligence** - Parameter_Referenc_Date_Type controls role-playing dimension selection
- **ERD #3: Field Measurements & Time Series** - Parameter_Reading_Slicer, Lkp_Aggregation_Type, TimeSeries_Filtered_1/2
- **ERD #5: User, Team & Security** - RLS tables integrate with Dim_User_Reference and Dim_Teams
- **ERD #6: Templates, Fragments & Configuration** - WorkTemplate_Filtered_1/2 filter Dim_WorkTemplates
- **All ERDs** - _Measures table contains measures that span all functional areas

### Table Documentation
- `tables/_Measures.md` - Central DAX measures table with 200+ measures
- `tables/Parameter_Asset_Hours_Type.md` - Asset hours parameter table
- `tables/Lkp_Shift_Duration.md` - Shift duration lookup
- `tables/Lkp_Label_Alias.md` - Label localization table
- `tables/Dim_Is_Completed.md` - Completion status boolean helper
- `tables/Semantic_Model_Tables.md` - Table metadata
- `tables/Semantic_Model_Columns.md` - Column metadata
- `tables/Semantic_Model_Relationships.md` - Relationship metadata
- `tables/RLS_Roles.md` - RLS role definitions
- `tables/Last_Refreshed_UTC.md` - Refresh timestamp table
- `tables/Organisation_Info.md` - Organisation branding table

---

## Change History

| Date | Change Description | Modified By |
|------|-------------------|-------------|
| 2025-11-18 | Initial ERD documentation created from TMDL metadata | AI Documentation Generator |
