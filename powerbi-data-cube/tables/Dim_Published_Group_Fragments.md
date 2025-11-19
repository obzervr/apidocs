# Dim_Published_Group_Fragments

**Table Type:** Dimension Table

**Purpose:** Published group fragment definitions from template hierarchy. Groups represent the top level of the three-tier fragment structure (Group → Section → Field).

**Last Updated:** November 19, 2025

---

## Overview

Dim_Published_Group_Fragments contains the published group fragment definitions that form the highest level of the template hierarchy in Obzervr. Groups are containers that organize sections and fields within work templates, and they support multiple instances (time series pattern), parent-child hierarchies, and various behavioral flags that control how data is captured and displayed.

A "Group Fragment" can be thought of as a reusable data collection template that defines:
- The structure of a repeatable data entry form
- Whether it can have multiple instances (e.g., multiple work requests per inspection)
- Whether it supports time-based tracking
- How it behaves in the Obzervr mobile and web applications

---

## Columns

| Column Name | Data Type | Description |
|------------|-----------|-------------|
| **Group_Id** | String | Group fragment identifier (Primary Key) |
| **Tenant_Id** | String | Tenant identifier for multi-tenancy |
| **Template_Link** | String | Reference to parent template |
| **Identifier** | String | Internal group identifier |
| **Display_Name** | String | Display name for the group |
| **Name** | String | Group name |
| **Description** | String | Group description text |
| **Is_SingleInstance** | Boolean | Single instance flag (false = multiple instances allowed) |
| **Is_TimeBased** | Boolean | Time-based tracking flag |
| **Is_Templated** | Boolean | Templated flag |
| **Archived** | Boolean | Archived status |
| **Has_NamedReoccurences** | Boolean | Named reoccurrences flag |
| **Allow_Uncomplete** | Boolean | Allow uncomplete flag |
| **Is_Preliminary** | Boolean | Preliminary status flag |
| **Has_Priority** | Boolean | Priority flag |
| **Is_Optional** | Boolean | Optional group flag |
| **Is_EntryGroup** | Boolean | Entry group flag |
| **Is_CodeEnabled** | Boolean | Code enabled flag |
| **Is_ListingGroup** | Boolean | Listing group flag |
| **Is_Published** | Boolean | Published status flag |
| **Is_Draft** | Boolean | Draft status flag |
| **Version** | String | Version string |
| **Version_Id** | String | Version identifier |
| **Published_At** | DateTime | Publication timestamp (Format: dd/MM/yyyy) |
| **Created_Date** | DateTime | Creation timestamp |
| **Last_Updated** | DateTime | Last update timestamp |
| **Fragment_Type** | String | Fragment type classification (typically "GroupFragment") |
| **Fragment_URL** | String | Fragment URL (DataCategory: WebUrl) |

---

## Key Boolean Flags

### Core Behavior Flags

**Is_SingleInstance**
- **True:** Only one instance of this group allowed per assignment
- **False:** Multiple instances allowed (time series pattern)
- **Example Use Case:** Pre-start checks (single) vs. Work notifications (multiple)

**Is_TimeBased**
- **True:** Group captures time-stamped data for time series analysis
- **False:** Group captures point-in-time snapshot data
- **Example Use Case:** Hourly readings (time-based) vs. Inspection checklist (non-time-based)

**Is_Templated**
- **True:** Group follows a template structure
- **False:** Free-form group
- **Example Use Case:** Structured inspections (templated) vs. Ad-hoc notes (non-templated)

### Status and Configuration Flags

**Archived**
- **True:** Group is archived and not available for new assignments
- **False:** Group is active
- **Purpose:** Soft delete pattern for historical data retention

**Is_Published** / **Is_Draft**
- **Is_Published = True:** Group is published and available for use
- **Is_Draft = True:** Group is in draft state and not yet published
- **Pattern:** Typically mutually exclusive (published XOR draft)

**Is_Optional**
- **True:** Group completion is optional in assignments
- **False:** Group completion is required
- **Purpose:** Flexibility in required vs. optional data collection

### Advanced Behavior Flags

**Has_NamedReoccurences**
- **True:** Each instance has a unique name/identifier
- **False:** Instances are numbered sequentially
- **Example:** Named equipment checks vs. numbered observations

**Allow_Uncomplete**
- **True:** Group can be marked complete even if not all fields filled
- **False:** All required fields must be completed
- **Purpose:** Flexibility for partial data capture scenarios

**Is_Preliminary**
- **True:** Group is preliminary/provisional
- **False:** Group is final
- **Purpose:** Draft data entry before final submission

**Has_Priority**
- **True:** Group has priority levels (e.g., high/medium/low)
- **False:** No priority classification
- **Purpose:** Work prioritization and triage

**Is_EntryGroup**
- **True:** Group is an entry point for data capture
- **False:** Group is internal/sub-group
- **Purpose:** UI rendering and navigation hints

**Is_CodeEnabled**
- **True:** Group supports coding/categorization
- **False:** No coding available
- **Purpose:** Integration with external systems or classification schemes

**Is_ListingGroup**
- **True:** Group displays as a list view
- **False:** Group displays as form view
- **Purpose:** UI rendering preference

---

## Relationships

### Outgoing Relationships

**To Fact_Fragment_Details** (One-to-Many)
- **Relationship ID:** 268af7b9-1856-d4a7-0be0-77b7e680e462
- **From Column:** Dim_Published_Group_Fragments[Group_Id]
- **To Column:** Fact_Fragment_Details[Group_Id]
- **Purpose:** Snowflake relationship linking group fragments to detailed specifications. Enables drill-down from group to its sections and fields.

**To Dim_Fragment_Type** (Many-to-One, Inactive)
- **Relationship ID:** AutoDetected_862df2ed-8596-413b-b180-585fde0873e2
- **From Column:** Dim_Published_Group_Fragments[Fragment_Type]
- **To Column:** Dim_Fragment_Type[Fragment_Type]
- **Active:** False
- **Purpose:** Role-playing dimension for group fragment type filtering via USERELATIONSHIP. Typically Fragment_Type = "GroupFragment".

### Incoming Relationships

**From Dim_Fragments** (Many-to-One)
- **Relationship ID:** 909dc464-ba0d-6bf3-0935-bc2a7c33e424
- **From Column:** Dim_Fragments[TemplateLink]
- **To Column:** Dim_Published_Group_Fragments[Template_Link]
- **Purpose:** Links master fragment catalog to published group fragments via template link.

---

## Data Source

**Source Type:** SQL Query

**Source Table/View:** VW_TemplateGroupFragments

**Transformation Logic:**
```sql
SELECT 
    GroupId AS Group_Id,
    TenantId AS Tenant_Id,
    TemplateLink AS Template_Link,
    Identifier,
    DisplayName AS Display_Name,
    Name,
    Description,
    IsSingleInstance AS Is_SingleInstance,
    IsTimeBased AS Is_TimeBased,
    IsTemplated AS Is_Templated,
    Archived,
    HasNamedReoccurences AS Has_NamedReoccurences,
    AllowUncomplete AS Allow_Uncomplete,
    IsPreliminary AS Is_Preliminary,
    HasPriority AS Has_Priority,
    IsOptional AS Is_Optional,
    IsEntryGroup AS Is_EntryGroup,
    IsCodeEnabled AS Is_CodeEnabled,
    IsListingGroup AS Is_ListingGroup,
    IsPublished AS Is_Published,
    IsDraft AS Is_Draft,
    Version,
    VersionId AS Version_Id,
    PublishedAt AS Published_At,
    CreatedDate AS Created_Date,
    LastUpdated AS Last_Updated,
    FragmentType AS Fragment_Type,
    FragmentURL AS Fragment_URL
FROM VW_TemplateGroupFragments
WHERE ([Tenant filtering logic])
```

**Key Transformations:**
1. Column renaming for Power BI conventions
2. Tenant filtering based on AllTenants or specific TenantId1-5 parameters
3. Published_At formatted as dd/MM/yyyy
4. Fragment_URL categorized as WebUrl

---

## Common DAX Patterns

### Count Published Group Fragments
```dax
Published Group Fragment Count = 
CALCULATE(
    COUNTROWS(Dim_Published_Group_Fragments),
    Dim_Published_Group_Fragments[Is_Published] = TRUE(),
    Dim_Published_Group_Fragments[Archived] = FALSE()
)
```

### Group Fragments by Template
```dax
EVALUATE
SUMMARIZE(
    Dim_Published_Group_Fragments,
    Dim_Published_Group_Fragments[Template_Link],
    "Group Count", COUNTROWS(Dim_Published_Group_Fragments)
)
ORDER BY [Group Count] DESC
```

### Multi-Instance Groups
```dax
Multi Instance Groups = 
CALCULATE(
    COUNTROWS(Dim_Published_Group_Fragments),
    Dim_Published_Group_Fragments[Is_SingleInstance] = FALSE()
)
```

### Time-Based Group Count
```dax
Time Based Groups = 
CALCULATE(
    COUNTROWS(Dim_Published_Group_Fragments),
    Dim_Published_Group_Fragments[Is_TimeBased] = TRUE()
)
```

### Optional vs Required Groups
```dax
Optional Group Percentage = 
VAR OptionalCount = 
    CALCULATE(
        COUNTROWS(Dim_Published_Group_Fragments),
        Dim_Published_Group_Fragments[Is_Optional] = TRUE()
    )
VAR TotalCount = COUNTROWS(Dim_Published_Group_Fragments)
RETURN
    DIVIDE(OptionalCount, TotalCount, 0) * 100
```

### Groups with Sections/Fields
```dax
Group with Details = 
ADDCOLUMNS(
    Dim_Published_Group_Fragments,
    "Section Count", CALCULATE(
        DISTINCTCOUNT(Fact_Fragment_Details[Section_Id]),
        ALLEXCEPT(Fact_Fragment_Details, Fact_Fragment_Details[Group_Id])
    ),
    "Field Count", CALCULATE(COUNTROWS(Fact_Fragment_Details))
)
```

---

## Understanding the Fragment Hierarchy

### Three-Tier Structure

**Level 1: Group (This Table)**
- Highest level container in template
- Defines repeatable data collection structure
- Can have multiple instances (if Is_SingleInstance = FALSE)
- Example: "Pre-Start Checks", "Work Notifications", "Safety Observations"

**Level 2: Section (Dim_Published_Section_Fragments)**
- Mid-level container within group
- Organizes related fields
- Example: "Engine Checks", "Fluid Levels", "Tire Inspection"

**Level 3: Field (Dim_Published_Field_Fragments)**
- Lowest level - individual data capture point
- Actual measurement, text input, photo, etc.
- Example: "Oil Pressure (PSI)", "Coolant Level (Liters)", "Left Front Tire Photo"

### Real-World Example: Mining Truck Pre-Start Inspection

**Group:** "Pre-Start Inspection" (Is_SingleInstance = TRUE)
- **Section:** "Engine Checks"
  - **Field:** Oil Level (Numeric)
  - **Field:** Oil Condition (Dropdown: Good/Fair/Poor)
  - **Field:** Engine Photo (Photo)
- **Section:** "Tire Inspection"
  - **Field:** Left Front Pressure (Numeric)
  - **Field:** Left Front Condition (Dropdown)
  - **Field:** Right Front Pressure (Numeric)
  - **Field:** Right Front Condition (Dropdown)

**Group:** "Work Notifications" (Is_SingleInstance = FALSE - multiple instances)
- **Section:** "Notification Details"
  - **Field:** Issue Description (Text)
  - **Field:** Severity (Dropdown: Low/Medium/High)
  - **Field:** Photo Evidence (Photo)
  - **Field:** Recommended Action (Text)

---

## Data Quality Considerations

### Referential Integrity
- **Group_Id:** Must be unique (Primary Key)
- **Template_Link:** Should exist in Dim_Published_Worktemplates
- **Fragment_Type:** Should = "GroupFragment" for this table

### Boolean Flag Validation
- **Is_Published and Is_Draft:** Should be mutually exclusive (one TRUE, other FALSE)
- **Archived:** Archived groups should have Is_Published = FALSE
- **Is_SingleInstance:** Must have clear business definition per group

### Version Management
- **Version and Version_Id:** Should be consistent and incrementing
- **Published_At:** Should not be BLANK for published fragments
- **Last_Updated:** Should be >= Created_Date

---

## Performance Considerations

### Query Optimization
- Filter by Is_Published and Archived early in queries
- Use Group_Id for joins (indexed primary key)
- Avoid scanning all boolean flags unnecessarily

### Indexing Recommendations (Source Database)
- Primary Key: Group_Id
- Foreign Key: Template_Link
- Filter Columns: Is_Published, Archived, Fragment_Type

---

## Related Tables

### Parent Templates
- **Dim_Published_Worktemplates** - Parent work templates
- **Dim_WorkTemplates** - Full template catalog

### Fragment Hierarchy
- **Dim_Published_Section_Fragments** - Sections within groups (next level down)
- **Dim_Published_Field_Fragments** - Fields within sections (lowest level)
- **Fact_Fragment_Details** - Detailed fragment specifications (snowflake fact)

### Fragment Catalog
- **Dim_Fragments** - Master fragment registry with versioning

### Lookup Tables
- **Dim_Fragment_Type** - Fragment type classification

---

## Data Model Position

**Related ERDs:**
- **ERD #6: Templates, Fragments & Configuration** - Primary documentation
- **ERD #3: Field Measurements & Time Series** - Groups become TimeSeries records in assignments

**Model Layer:** Dimension (Snowflake Schema)

**Refresh Frequency:** Typically low (daily or on-demand when templates change)

---

## Change History

| Date | Change Description | Modified By |
|------|-------------------|-------------|
| 2025-11-19 | Initial table documentation created | AI Documentation Generator |

---

## Additional Notes

### Group vs. TimeSeries Distinction
In operational data:
- **Dim_Published_Group_Fragments** defines the template/structure
- **TimeSeries** table contains actual instances when groups are used in assignments
- **TimeSeries.Group_Fragment_Reference** links back to this table's Group_Id

Example:
- **Template:** "Truck Pre-Start" has Group "Work Notifications"
- **Assignment:** "Truck 405 Pre-Start" creates TimeSeries instances for each notification
- Each TimeSeries record references Group_Id from this table

### Boolean Flag Combinations
Common combinations:
- **Standard Group:** Is_SingleInstance=TRUE, Is_Optional=FALSE, Is_Published=TRUE
- **Time Series Group:** Is_SingleInstance=FALSE, Is_TimeBased=TRUE, Has_NamedReoccurences=TRUE
- **Optional Checklist:** Is_SingleInstance=TRUE, Is_Optional=TRUE, Allow_Uncomplete=TRUE

### Fragment URL Usage
Fragment_URL provides deep linking to fragment designer in Obzervr application for template administration and editing.
