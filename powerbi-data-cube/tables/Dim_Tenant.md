# Dim_Tenant

**Table Type:** Dimension Table

**Purpose:** Tenant (organization/customer) dimension providing tenant settings, branding configuration, timezone settings, and custom UI labels for multi-tenant environments.

**Last Updated:** November 19, 2025

---

## Overview

Dim_Tenant is the central tenant configuration dimension that enables multi-tenancy across the Obzervr platform. Each tenant represents a separate customer, organization, or business unit with isolated data, customized branding, timezone-specific time intelligence, and configurable UI terminology. This table serves as the foundation for tenant-based filtering and Row-Level Security (RLS) throughout the semantic model.

---

## Columns

| Column Name | Data Type | Description | Characteristics |
|------------|-----------|-------------|-----------------|
| **Tenant_Id** | String | Tenant identifier (Primary Key) | PK, Not Null |
| **Tenant_Code** | String | Short tenant code | Used in display names |
| **Timezone** | String | Tenant timezone (e.g., "Australia/Sydney") | Used for time calculations |
| **Logo_Base64** | String | Base64 encoded tenant logo | Hidden, DataCategory: ImageUrl |
| **Tenant_Name** | String | Full tenant name | Display-friendly |
| **Tenant_URL** | String | Tenant application URL | DataCategory: WebUrl |
| **Offset_Minutes** | String | Timezone offset in minutes from UTC | Calculated from Timezone |
| **Primary_Colour** | String | Primary branding colour (hex) | UI customization |
| **Secondary_Colour** | String | Secondary branding colour (hex) | UI customization |
| **Assignment_PointList_1** | String | Custom label for assignment point list 1 | Comma-delimited list |
| **Assignment_Point_List_2** | String | Custom label for assignment point list 2 | Comma-delimited list |
| **Template_List_1** | String | Custom label for template list 1 | Comma-delimited list |
| **Template_List_2** | String | Custom label for template list 2 | Comma-delimited list |
| **Series_List_1** | String | Custom label for series list 1 | Comma-delimited list |
| **Series_List_2** | String | Custom label for series list 2 | Comma-delimited list |
| **OrganisationKey** | String | Organisation key identifier | Integration key |
| **Tenant** | String | Calculated: Tenant_Code & " - " & Tenant_Name | Display label |

---

## Calculated Columns

### Tenant (Display Label)
```dax
Tenant = Dim_Tenant[Tenant_Code] & " - " & Dim_Tenant[Tenant_Name]
```
**Purpose:** Provides user-friendly display label combining code and name (e.g., "ACME - Acme Corporation")

### Offset_Minutes (Timezone Conversion)
**Purpose:** Converts timezone string to numeric offset from UTC in minutes

**Source:** Derived from `Fn_ConvertTimezonetoOffsetMinutes` function applied to Timezone column

**Example Values:**
- "Australia/Sydney" → +600 minutes (UTC+10)
- "America/New_York" → -300 minutes (UTC-5)
- "Europe/London" → 0 minutes (UTC)

**Usage:** Critical for time intelligence calculations to convert UTC timestamps to local tenant time

---

## Column Categories

### Data Categories
- **Logo_Base64:** ImageUrl (enables image rendering in Power BI)
- **Tenant_URL:** WebUrl (enables clickable links in reports)

### Hidden Columns
- **Logo_Base64:** Hidden from field list (use in dedicated visual, not general browsing)

---

## Relationships

### Outgoing Relationships (Many-to-One from Fact/Dimension Tables)

**From Assignments**
- **Relationship ID:** (Various - see ERD #1)
- **From Column:** Assignments[Tenant_Id]
- **To Column:** Dim_Tenant[Tenant_Id]
- **Purpose:** Multi-tenant isolation for assignment records

**From Dim_Shifts**
- **Relationship ID:** 102
- **From Column:** Dim_Shifts[Tenant_Id]
- **To Column:** Dim_Tenant[Tenant_Id]
- **Purpose:** Tenant-specific shift schedules

**From Dim_Shift_Time_FromDate**
- **Relationship ID:** 116
- **From Column:** Dim_Shift_Time_FromDate[Tenant_Id]
- **To Column:** Dim_Tenant[Tenant_Id]
- **Purpose:** Tenant-specific shift time definitions

**From Dim_Shift_Time_CompletedOn**
- **Relationship ID:** 120
- **From Column:** Dim_Shift_Time_CompletedOn[Tenant_Id]
- **To Column:** Dim_Tenant[Tenant_Id]
- **Purpose:** Tenant-specific shift time for completion analysis

**From TimeSeries**
- **Relationship ID:** 124
- **From Column:** TimeSeries[Tenant_Id]
- **To Column:** Dim_Tenant[Tenant_Id]
- **Purpose:** Multi-tenant isolation for time series records

**From TimeSeries_FieldMeasurements**
- **Relationship ID:** 128
- **From Column:** TimeSeries_FieldMeasurements[Tenant_Id]
- **To Column:** Dim_Tenant[Tenant_Id]
- **Purpose:** Multi-tenant isolation for field measurements

**From Fact_User_Audit_Command_Count_By_Day**
- **Relationship ID:** AutoDetected_28d59c09-f648-4e1c-9547-5535797889dd
- **From Column:** Fact_User_Audit_Command_Count_By_Day[Tenant_Id]
- **To Column:** Dim_Tenant[Tenant_Id]
- **Purpose:** Tenant-specific usage analytics

**From Dim_Worklist_Views**
- **Relationship ID:** c3be2bf1-d131-e98f-cb35-cc64d24f4be3
- **From Column:** Dim_Worklist_Views[Tenant_Id]
- **To Column:** Dim_Tenant[Tenant_Id]
- **Purpose:** Tenant-specific worklist view configurations

**From Fact_Fragment_Details**
- **Relationship ID:** AutoDetected_c1ce6178-d503-4aae-8e51-d6b6e70ec068
- **From Column:** Fact_Fragment_Details[Tenant_Id]
- **To Column:** Dim_Tenant[Tenant_Id]
- **Purpose:** Tenant-specific template fragment details

**Additional Relationships:** Multiple other fact and dimension tables connect to Dim_Tenant via Tenant_Id

---

## Data Source

**Source Type:** SQL Query

**Source Table/View:** TenantSettings

**Transformation Logic:**
```sql
SELECT 
    TenantId AS Tenant_Id,
    TenantCode AS Tenant_Code,
    Timezone,
    LogoBase64 AS Logo_Base64,
    TenantName AS Tenant_Name,
    TenantURL AS Tenant_URL,
    Fn_ConvertTimezonetoOffsetMinutes(Timezone) AS Offset_Minutes,
    PrimaryColour AS Primary_Colour,
    SecondaryColour AS Secondary_Colour,
    AssignmentPointList1 AS Assignment_PointList_1,
    AssignmentPointList2 AS Assignment_Point_List_2,
    TemplateList1 AS Template_List_1,
    TemplateList2 AS Template_List_2,
    SeriesList1 AS Series_List_1,
    SeriesList2 AS Series_List_2,
    OrganisationKey
FROM TenantSettings
WHERE ([Tenant filtering logic based on AllTenants or TenantId1-5 parameters])
```

**Key Transformations:**
1. Column renaming for Power BI conventions
2. Timezone offset calculation via Fn_ConvertTimezonetoOffsetMinutes UDF
3. Tenant filtering based on semantic model parameters
4. Calculated Tenant column added in DAX

---

## Key Usage Patterns

### 1. Multi-Tenant Data Isolation

**Pattern:** All fact tables include Tenant_Id with relationship to Dim_Tenant

**RLS Implementation:**
```dax
// RLS Rule on Dim_Tenant table
VAR CurrentUser = USERPRINCIPALNAME()
VAR UserTenants = 
    CALCULATETABLE(
        VALUES(RLS_Users[Tenant_Id]),
        RLS_Users[Email] = CurrentUser
    )
RETURN
    Dim_Tenant[Tenant_Id] IN UserTenants
```

**Benefits:**
- Single semantic model serves multiple customers
- Complete data isolation between tenants
- Scalable multi-tenancy without model duplication

---

### 2. Timezone-Aware Time Intelligence

**Pattern:** Use Offset_Minutes to convert UTC to tenant local time

**Example: Current Local Time**
```dax
Current Local Time = 
UTCNOW() + TIME(0, MAX(Dim_Tenant[Offset_Minutes]), 0)
```

**Example: Convert UTC Timestamp to Local**
```dax
Assignment Local Start Time = 
Assignments[From_Date] + TIME(0, RELATED(Dim_Tenant[Offset_Minutes]), 0)
```

**Example: Relative Date Calculation with Timezone**
```dax
Today (Tenant Local) = 
VAR LocalNow = UTCNOW() + TIME(0, MAX(Dim_Tenant[Offset_Minutes]), 0)
RETURN DATEVALUE(LocalNow)
```

---

### 3. Dynamic Branding

**Pattern:** Use Primary_Colour, Secondary_Colour, and Logo_Base64 for tenant-specific branding

**Example: Dynamic Title Color**
```dax
Title Color = MAX(Dim_Tenant[Primary_Colour])
```

**Example: Logo Display**
- Use Logo_Base64 directly in Image visual
- Data category ImageUrl enables automatic rendering

**Example: Branded URL**
```dax
Tenant Portal Link = MAX(Dim_Tenant[Tenant_URL]) & "/portal"
```

---

### 4. Custom UI Labels (Filtered Dimensions)

**Pattern:** Use comma-delimited lists to create filtered dimension tables

**Template_List_1 Usage:**
- Source: "Template A, Template B, Template C"
- Transformed to: WorkTemplate_Filtered_1 table with 3 rows
- Enables tenant-specific template slicers

**Example M Code (WorkTemplate_Filtered_1):**
```m
let
    Source = Dim_Tenant,
    #"Split Column" = Table.ExpandListColumn(
        Table.TransformColumns(
            Source, 
            {{"Template_List_1", Splitter.SplitTextByDelimiter(",", QuoteStyle.Csv)}}
        ), 
        "Template_List_1"
    ),
    #"Renamed" = Table.RenameColumns(#"Split Column", {{"Template_List_1", "WorkTemplate_Name"}})
in
    #"Renamed"
```

---

## Common DAX Patterns

### Get Current Tenant Information
```dax
Current Tenant Name = 
SELECTEDVALUE(Dim_Tenant[Tenant_Name], "All Tenants")
```

### Filter by Tenant Code
```dax
ACME Assignments = 
CALCULATE(
    COUNTROWS(Assignments),
    Dim_Tenant[Tenant_Code] = "ACME"
)
```

### List All Tenants
```dax
Tenant List = 
CONCATENATEX(
    Dim_Tenant,
    Dim_Tenant[Tenant],
    ", ",
    Dim_Tenant[Tenant_Code],
    ASC
)
```

### Check if Multi-Tenant Selection
```dax
Is Multi Tenant Selection = 
COUNTROWS(VALUES(Dim_Tenant[Tenant_Id])) > 1
```

### Tenant-Specific Date Range
```dax
Tenant Local Date Range = 
VAR MinUTC = MIN(Assignments[From_Date])
VAR MaxUTC = MAX(Assignments[From_Date])
VAR OffsetMins = MAX(Dim_Tenant[Offset_Minutes])
RETURN
    FORMAT(MinUTC + TIME(0, OffsetMins, 0), "dd/MM/yyyy") & " to " &
    FORMAT(MaxUTC + TIME(0, OffsetMins, 0), "dd/MM/yyyy")
```

---

## Tenant Configuration Guide

### Branding Configuration

**Primary_Colour / Secondary_Colour Format:**
- Hex color codes (e.g., "#0078D4", "#F25022")
- Include # prefix
- Used in conditional formatting and dynamic titles

**Logo_Base64 Format:**
- Base64 encoded image string
- Supports PNG, JPEG, GIF formats
- Recommended size: 200x200 pixels or smaller
- Example: `data:image/png;base64,iVBORw0KGg...`

**Tenant_URL Format:**
- Full URL including protocol (e.g., "https://acme.obzervr.com")
- Used for deep linking and portal navigation
- Should point to tenant-specific application instance

---

### Custom List Configuration

**Assignment_PointList_1/2, Template_List_1/2, Series_List_1/2:**
- Comma-delimited strings
- No quotes around individual items
- Example: `"Template A, Template B, Template C"`
- Transformed into filtered dimension tables (e.g., WorkTemplate_Filtered_1)
- Used to limit slicer options to tenant-relevant values

**Best Practices:**
- Keep lists concise (avoid hundreds of items)
- Use consistent naming conventions
- Update lists when adding/removing templates or points
- Document list purposes (List_1 vs List_2)

---

### Timezone Configuration

**Supported Timezone Formats:**
- IANA timezone database names (e.g., "Australia/Sydney", "America/New_York")
- Must be recognized by Fn_ConvertTimezonetoOffsetMinutes function
- Handles daylight saving time transitions automatically

**Common Timezones:**
- `"Australia/Sydney"` → UTC+10/+11 (DST)
- `"America/New_York"` → UTC-5/-4 (DST)
- `"Europe/London"` → UTC+0/+1 (DST)
- `"Asia/Singapore"` → UTC+8 (no DST)
- `"UTC"` → UTC+0

---

## Related Tables

### Row-Level Security
- **RLS_Users** - User-tenant associations for RLS filtering
- **RLS_Tenant_User_Analytic** - Analytics access by tenant

### Filtered Dimensions (Derived from Dim_Tenant)
- **WorkTemplate_Filtered_1** - Template List 1
- **WorkTemplate_Filtered_2** - Template List 2
- **Subsite_Filtered_1** - Assignment Point List 1
- **Subsite_Filtered_2** - Assignment Point List 2
- **TimeSeries_Filtered_1** - Series List 1
- **TimeSeries_Filtered_2** - Series List 2

### Configuration
- **Organisation_Info** - Calculated table derived from Dim_Tenant (Logo_Base64, OrganisationKey)

### Fact Tables (Examples)
- **Assignments** - Work orders by tenant
- **TimeSeries** - Data collection groups by tenant
- **TimeSeries_FieldMeasurements** - Field measurements by tenant
- **Fact_User_Audit_Command_Count_By_Day** - Usage analytics by tenant

---

## Data Model Position

**Related ERDs:**
- **ERD #5: User, Team & Security** - Primary documentation for multi-tenancy
- **ERD #1: Assignment Core Model** - Tenant filtering for assignments
- **ERD #2: Date Dimensions & Time Intelligence** - Timezone offset usage
- **ERD #3: Field Measurements & Time Series** - Tenant isolation for measurements
- **ERD #6: Templates, Fragments & Configuration** - Custom list filtering
- **ERD #7: Fact Tables & Audit** - Tenant-specific audit tracking

**Model Layer:** Core Dimension

**Refresh Frequency:** Low (typically daily or on-demand when configuration changes)

---

## Change History

| Date | Change Description | Modified By |
|------|-------------------|-------------|
| 2025-11-19 | Initial table documentation created | AI Documentation Generator |

---

## Additional Notes

### Multi-Tenant Semantic Model Strategy
Dim_Tenant enables a **unified multi-tenant semantic model** strategy:
- **Single Model:** One semantic model serves all tenants
- **Row-Level Security:** Tenant_Id filtering ensures data isolation
- **Customization:** Branding and labels personalize experience per tenant
- **Scalability:** Add new tenants without redeploying semantic model
- **Maintenance:** Updates apply to all tenants simultaneously

### Alternative Strategy (Multiple Models)
For comparison, a **separate semantic model per tenant** approach:
- Pros: Complete isolation, simpler security, dedicated capacity
- Cons: Higher maintenance, duplicate logic, slower updates, capacity inefficiency

The unified model (leveraging Dim_Tenant) is preferred for SaaS/multi-customer scenarios.

### Timezone Offset Handling
The Offset_Minutes calculation handles:
- **Static Offsets:** UTC offset for standard time
- **Daylight Saving Time:** Fn_ConvertTimezonetoOffsetMinutes adjusts for DST
- **Dynamic Calculation:** Refresh captures current DST state
- **Limitation:** Historical DST transitions not captured (uses current offset)

For precise historical DST handling, consider date-specific offset lookup tables.
