# Dim_Subsites

**Table Type:** Dimension Table (Legacy)  
**Schema:** dbo.DimSubSites  
**Primary Key:** SubSite_Id  
**Related ERD:** [ERD #1: Assignment Core Model](../ERDs/ERD_01_Assignment_Core.md)

---

## Table Overview

Contains information about subsites (child locations of main sites), including location details, address, coordinates, usage, and tenant-specific filtering flags. This table represents the second level of the legacy fixed hierarchy (Site → Subsite) which has been deprecated in favor of the flexible Dim_AssignmentPoints hierarchy.

**Current Status:** Legacy table - use Dim_AssignmentPoints for new implementations

**Source System:** Analytic Database (dbo.DimSubSites)

**Parent Relationship:** Links to Dim_Sites via Parent_SiteI_d

**Row Count:** Varies by tenant (typically 10-500 subsites per organization)

**Refresh Type:** Full refresh with tenant filtering

**Multi-Tenant:** Yes (filtered by Tenant_Id)

---

## Table Specifications

| Property | Value |
|----------|-------|
| Lineage Tag | 39eacdf0-2df5-422d-bb96-08dc04d2cf48 |
| Query Group | Dimensions |
| Partitions | 1 (Dim_Subsites) |
| Relationships | 1 outbound to Dim_Tenant, 1 outbound to Dim_Sites (inferred) |
| Calculated Columns | 2 (Is_Subsite_Filtered1, Is_Subsite_Filtered2) |
| Hierarchies | 0 |

---

## Column Specifications

| Column Name | Data Type | Description | Source Column | Key | Nullable |
|-------------|-----------|-------------|---------------|-----|----------|
| **Tenant_Id** | string | Tenant identifier | TenantId | FK | No |
| **Id** | string | Internal record identifier | Id | - | No |
| **SubSite_Id** | string | Unique subsite identifier | SubSiteId | PK | No |
| **SubSite_Name** | string | Display name of subsite | SubSiteName | - | Yes |
| **SubSite_AddressLine1** | string | Address line 1 | SubSiteAddressLine1 | - | Yes |
| **SubSite_AddressLine2** | string | Address line 2 | SubSiteAddressLine2 | - | Yes |
| **SubSite_AddressLine3** | string | Address line 3 | SubSiteAddressLine3 | - | Yes |
| **SubSite_AddressCity** | string | City | SubSiteAddressCity | - | Yes |
| **SubSite_AddressPostCode** | string | Postal code | SubSiteAddressPostCode | - | Yes |
| **SubSite_LocationDescription** | string | Location description | SubSiteLocationDescription | - | Yes |
| **SubSite_UsageDescription** | string | Usage description | SubSiteUsageDescription | - | Yes |
| **SubSite_Latitude** | double | Latitude coordinate | APLatitude | - | Yes |
| **SubSite_Longitude** | double | Longitude coordinate | APLongitude | - | Yes |
| **Last_Updated** | datetime | Last update timestamp | LastUpdated | - | Yes |
| **Created_Date** | datetime | Creation timestamp | CreatedDate | - | Yes |
| **Is_Deleted** | boolean | Soft delete flag | IsDeleted | - | No |
| **Parent_SiteI_d** | string | Parent site identifier | ParentSiteId | FK | Yes |
| **Is_Subsite_Filtered1** | int | Filter set 1 membership flag (calculated) | - | - | No |
| **Is_Subsite_Filtered2** | int | Filter set 2 membership flag (calculated) | - | - | No |
| **LastLoaded** | datetime | ETL load timestamp | LastLoaded | - | Yes |

---

## Calculated Columns

### Is_Subsite_Filtered1

**Purpose:** Flag indicating whether the subsite is included in tenant-specific Subsite_Filtered_1 list

**Data Type:** Integer (0 or 1)

**DAX Expression:**
```dax
Is_Subsite_Filtered1 = 
IF (
    ISBLANK (
        LOOKUPVALUE(
            Subsite_Filtered_1[Assignment_Point_Id],
            Subsite_Filtered_1[Assignment_Point_Id],
            Dim_Subsites[SubSite_Id]
        )
    ),
    0,
    1
)
```

**Logic:**
- Performs LOOKUPVALUE to check if SubSite_Id exists in Subsite_Filtered_1 table
- Returns 1 if found (subsite is in filter list)
- Returns 0 if not found or BLANK (subsite not in filter list)

**Use Case:** Enables tenant-specific filtering where each tenant can define which subsites are included in Filter Set 1 (e.g., "Active Subsites", "North Region Subsites")

---

### Is_Subsite_Filtered2

**Purpose:** Flag indicating whether the subsite is included in tenant-specific Subsite_Filtered_2 list

**Data Type:** Integer (0 or 1)

**DAX Expression:**
```dax
Is_Subsite_Filtered2 = 
IF (
    ISBLANK (
        LOOKUPVALUE(
            Subsite_Filtered_2[Assignment_Point_Id],
            Subsite_Filtered_2[Assignment_Point_Id],
            Dim_Subsites[SubSite_Id]
        )
    ),
    0,
    1
)
```

**Logic:** Same as Is_Subsite_Filtered1 but checks Subsite_Filtered_2 table

**Use Case:** Second independent filter list for tenant-specific filtering (e.g., "South Region Subsites", "High Priority Subsites")

---

## Relationships

### Outbound Relationships

**To Dim_Tenant**
- **Type:** Many-to-one
- **From Column:** Tenant_Id
- **To Column:** Dim_Tenant[Tenant_Id]
- **Cardinality:** Many:1
- **Cross-filter Direction:** Single
- **Status:** Active
- **Purpose:** Links subsites to tenant configuration

**To Dim_Sites** (Inferred)
- **Type:** Many-to-one
- **From Column:** Parent_SiteI_d
- **To Column:** Dim_Sites[Site_Id]
- **Purpose:** Links subsites to parent sites in legacy hierarchy
- **Note:** May not be formally defined in semantic model relationships

---

## Power Query M Source

```m
let
    Source = Sql.Database(#"SQL Server", #"Analytic Database", [CreateNavigationProperties=false, MultiSubnetFailover=true]),
    Table = Source{[Schema="dbo",Item="DimSubSites"]}[Data],
    #"Filtered Rows" = Table.SelectRows(Table, each AllTenants = true or [TenantId] = #"TenantId1" or [TenantId] = TenantId2 or [TenantId] = TenantId3 or [TenantId] = TenantId4 or [TenantId] = TenantId5),
    #"Renamed Columns" = Table.RenameColumns(#"Filtered Rows",{
        {"TenantId", "Tenant_Id"}, 
        {"SubSiteId", "SubSite_Id"}, 
        {"SubSiteName", "SubSite_Name"}, 
        {"SubSiteAddressLine1", "SubSite_AddressLine1"}, 
        {"SubSiteAddressLine2", "SubSite_AddressLine2"}, 
        {"SubSiteAddressLine3", "SubSite_AddressLine3"}, 
        {"SubSiteAddressCity", "SubSite_AddressCity"}, 
        {"SubSiteAddressPostCode", "SubSite_AddressPostCode"}, 
        {"SubSiteLocationDescription", "SubSite_LocationDescription"}, 
        {"SubSiteUsageDescription", "SubSite_UsageDescription"}, 
        {"CreatedDate", "Created_Date"}, 
        {"APLatitude", "SubSite_Latitude"}, 
        {"APLongitude", "SubSite_Longitude"}, 
        {"LastUpdated", "Last_Updated"}, 
        {"IsDeleted", "Is_Deleted"}, 
        {"ParentSiteId", "Parent_SiteI_d"}
    })
in
    #"Renamed Columns"
```

**Key Transformations:**
1. SQL database connection with MultiSubnetFailover
2. Tenant filtering using parameters
3. Column renaming to snake_case
4. No soft delete filtering (Is_Deleted retained)

---

## DAX Query Patterns

### Subsites by parent site

```dax
EVALUATE
ADDCOLUMNS(
    FILTER(Dim_Subsites, [Is_Deleted] = FALSE()),
    "Subsite", [SubSite_Name],
    "Parent Site", LOOKUPVALUE(
        Dim_Sites[Site_Name],
        Dim_Sites[Site_Id],
        Dim_Subsites[Parent_SiteI_d]
    ),
    "Address", [SubSite_AddressLine1] & ", " & [SubSite_AddressCity],
    "In Filter 1", [Is_Subsite_Filtered1],
    "In Filter 2", [Is_Subsite_Filtered2]
)
ORDER BY [Parent Site], [Subsite]
```

### Subsites in filter set 1

```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Dim_Subsites,
        "Subsite", [SubSite_Name],
        "Usage", [SubSite_UsageDescription],
        "Tenant", RELATED(Dim_Tenant[Tenant]),
        "Is Deleted", [Is_Deleted]
    ),
    [Is_Subsite_Filtered1] = 1
)
ORDER BY [Tenant], [Subsite]
```

### Hierarchy report (Site → Subsite)

```dax
EVALUATE
ADDCOLUMNS(
    FILTER(Dim_Subsites, [Is_Deleted] = FALSE()),
    "Hierarchy", 
        LOOKUPVALUE(Dim_Sites[Site_Name], Dim_Sites[Site_Id], [Parent_SiteI_d]) 
        & " → " & [SubSite_Name],
    "Latitude", [SubSite_Latitude],
    "Longitude", [SubSite_Longitude]
)
ORDER BY [Hierarchy]
```

---

## Data Model Pattern

**Pattern:** Dimension Table (Legacy Fixed Hierarchy - Child Level)

**Characteristics:**
- **Legacy Status**: Deprecated in favor of Dim_AssignmentPoints
- **Fixed Hierarchy**: Child level in Site → Subsite hierarchy
- **Tenant-Specific Filtering**: Two calculated columns (Is_Subsite_Filtered1/2) enable dynamic filtering per tenant
- **Geographic Data**: Coordinates for mapping
- **Parent-Child Relationship**: Parent_SiteI_d links to Dim_Sites[Site_Id]

**Filtering Pattern:**
The Is_Subsite_Filtered1 and Is_Subsite_Filtered2 columns implement a flexible filtering mechanism:
1. Tenant configures comma-delimited lists in Dim_Tenant (Subsite_List_1, Subsite_List_2)
2. Power Query splits lists into Subsite_Filtered_1 and Subsite_Filtered_2 tables
3. DAX calculated columns in Dim_Subsites check membership via LOOKUPVALUE
4. Reports can filter by Is_Subsite_Filtered1 = 1 or Is_Subsite_Filtered2 = 1

This pattern allows different filter contexts without modifying the semantic model (configuration-driven filtering).

---

## Related Documentation

### ERD Documents
- [ERD #1: Assignment Core Model](../ERDs/ERD_01_Assignment_Core.md) - Documents legacy Site/Subsite hierarchy

### Related Tables
- **Dim_Sites** - Parent table (one-to-many from Dim_Sites perspective)
- **Subsite_Filtered_1** - Tenant-specific filter list 1
- **Subsite_Filtered_2** - Tenant-specific filter list 2
- **Dim_Tenant** - Tenant configuration with Subsite_List_1 and Subsite_List_2 columns
- **Dim_AssignmentPoints** - Replacement flexible hierarchy

### Other Documentation
- [ERD_Overview.md](../ERD_Overview.md) - Architecture patterns including filtered dimensions
- [Dim_Sites.md](Dim_Sites.md) - Parent table documentation

---

## Change History

| Date | Change Description | Modified By |
|------|-------------------|-------------|
| 2025-11-18 | Initial table documentation created from TMDL metadata | AI Documentation Generator |

---

## Notes

**Legacy Status**: Use Dim_AssignmentPoints for new implementations. This table maintained for backward compatibility.

**Filtering Flexibility**: The two filter flags enable tenant-specific filtering without semantic model changes. Tenants configure lists in Dim_Tenant, and the calculated columns automatically reflect membership.

**Coordinate Usage**: SubSite_Latitude and SubSite_Longitude enable mapping of individual subsites (more granular than site-level coordinates in Dim_Sites).

**Parent_SiteI_d Naming**: Note the unusual naming "Parent_SiteI_d" (capital I before lowercase d) - this appears to be a typo in the source column name but is preserved for consistency with the source system.
