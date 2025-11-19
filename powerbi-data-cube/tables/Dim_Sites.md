# Dim_Sites

**Table Type:** Dimension Table (Legacy)  
**Schema:** dbo.DimSites  
**Primary Key:** Site_Id  
**Related ERD:** [ERD #1: Assignment Core Model](../ERDs/ERD_01_Assignment_Core.md)

---

## Table Overview

Stores details of physical sites including name, address, coordinates, location description, and usage. This table represents the top level of the legacy fixed hierarchy (Site → Subsite) which has been deprecated in favor of the flexible Dim_AssignmentPoints hierarchy (Area → Location → Point). The table is retained for backward compatibility and historical reporting.

**Current Status:** Legacy table - use Dim_AssignmentPoints for new implementations

**Source System:** Analytic Database (dbo.DimSites)

**Row Count:** Varies by tenant (typically 1-50 sites per organization)

**Refresh Type:** Full refresh with tenant filtering

**Multi-Tenant:** Yes (filtered by Tenant_Id)

---

## Table Specifications

| Property | Value |
|----------|-------|
| Lineage Tag | cd96a0cb-3af7-4467-ab2f-2acea6b72e05 |
| Query Group | Dimensions |
| Partitions | 1 (Dim_Sites) |
| Relationships | 1 outbound to Dim_Tenant |
| Calculated Columns | 0 |
| Hierarchies | 0 |

---

## Column Specifications

| Column Name | Data Type | Description | Source Column | Key | Nullable |
|-------------|-----------|-------------|---------------|-----|----------|
| **Tenant_Id** | string | Tenant identifier for the site | TenantId | FK | No |
| **Id** | string | Internal record identifier | Id | - | No |
| **Site_Id** | string | Unique site identifier used for joins | SiteId | PK | No |
| **Site_Name** | string | Display name of the site | SiteName | - | Yes |
| **Site_AddressLine1** | string | Address line 1 | SiteAddressLine1 | - | Yes |
| **Site_AddressLine2** | string | Address line 2 | SiteAddressLine2 | - | Yes |
| **Site_AddressLine3** | string | Address line 3 | SiteAddressLine3 | - | Yes |
| **Site_AddressCity** | string | City of the site address | SiteAddressCity | - | Yes |
| **Site_AddressPostCode** | string | Postal/ZIP code | SiteAddressPostCode | - | Yes |
| **Site_LocationDescription** | string | Human-readable location description | SiteLocationDescription | - | Yes |
| **Site_UsageDescription** | string | Usage description or purpose of site | SiteUsageDescription | - | Yes |
| **Site_Latitude** | double | Latitude coordinate | APLatitude | - | Yes |
| **Site_Longitude** | double | Longitude coordinate | APLongitude | - | Yes |
| **Last_Updated** | datetime | Last update timestamp | LastUpdated | - | Yes |
| **Created_Date** | datetime | Creation timestamp | CreatedDate | - | Yes |
| **Is_Deleted** | boolean | Soft delete flag | IsDeleted | - | No |
| **LastLoaded** | datetime | ETL load timestamp | LastLoaded | - | Yes |

### Column Details

**Tenant_Id**
- Format: GUID string
- Purpose: Multi-tenant isolation
- Related: Dim_Tenant[Tenant_Id]

**Site_Id**
- Format: GUID string or custom identifier
- Purpose: Unique identifier for relationships
- Used by: Dim_Subsites[Parent_SiteI_d]

**Coordinates (Site_Latitude, Site_Longitude)**
- Format: Decimal degrees
- Purpose: Mapping and location services
- Example: -33.8688 (latitude), 151.2093 (longitude) for Sydney

**Is_Deleted**
- Values: TRUE (deleted), FALSE (active)
- Purpose: Soft delete pattern for data retention
- Note: Deleted records excluded via Power Query filter

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
- **Purpose:** Links sites to tenant configuration for multi-tenant filtering

### Inbound Relationships

**From Dim_Subsites**
- **Type:** One-to-many (from Dim_Sites perspective)
- **From Column:** Dim_Subsites[Parent_SiteI_d]
- **To Column:** Site_Id
- **Purpose:** Legacy parent-child hierarchy (Site → Subsite)
- **Note:** This relationship may not be formally defined in the semantic model as it's a legacy pattern

---

## Power Query M Source

```m
let
    Source = Sql.Database(#"SQL Server", #"Analytic Database", [CreateNavigationProperties=false, MultiSubnetFailover=true]),
    Table = Source{[Schema="dbo",Item="DimSites"]}[Data],
    #"Filtered Rows" = Table.SelectRows(Table, each AllTenants = true or [TenantId] = #"TenantId1" or [TenantId] = TenantId2 or [TenantId] = TenantId3 or [TenantId] = TenantId4 or [TenantId] = TenantId5),
    #"Renamed Columns" = Table.RenameColumns(#"Filtered Rows",{
        {"TenantId", "Tenant_Id"}, 
        {"SiteId", "Site_Id"}, 
        {"SiteName", "Site_Name"}, 
        {"SiteAddressLine1", "Site_AddressLine1"}, 
        {"SiteAddressLine2", "Site_AddressLine2"}, 
        {"SiteAddressLine3", "Site_AddressLine3"}, 
        {"SiteAddressCity", "Site_AddressCity"}, 
        {"SiteAddressPostCode", "Site_AddressPostCode"}, 
        {"SiteLocationDescription", "Site_LocationDescription"}, 
        {"SiteUsageDescription", "Site_UsageDescription"}, 
        {"APLatitude", "Site_Latitude"}, 
        {"APLongitude", "Site_Longitude"}, 
        {"CreatedDate", "Created_Date"}, 
        {"LastUpdated", "Last_Updated"}, 
        {"IsDeleted", "Is_Deleted"}
    })
in
    #"Renamed Columns"
```

**Key Transformations:**
1. SQL database connection with MultiSubnetFailover enabled
2. Tenant filtering using AllTenants or specific TenantId parameters (TenantId1-5)
3. Column renaming from PascalCase to snake_case convention
4. No soft delete filtering (Is_Deleted column retained for reporting)

---

## DAX Query Patterns

### List all active sites

```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Dim_Sites,
        "Site", [Site_Name],
        "City", [Site_AddressCity],
        "Latitude", [Site_Latitude],
        "Longitude", [Site_Longitude],
        "Usage", [Site_UsageDescription]
    ),
    [Is_Deleted] = FALSE()
)
ORDER BY [Site]
```

### Count subsites per site

```dax
EVALUATE
ADDCOLUMNS(
    FILTER(
        Dim_Sites,
        [Is_Deleted] = FALSE()
    ),
    "Site", [Site_Name],
    "Subsite Count", CALCULATE(
        COUNTROWS(Dim_Subsites),
        FILTER(
            Dim_Subsites,
            Dim_Subsites[Parent_SiteI_d] = Dim_Sites[Site_Id] &&
            Dim_Subsites[Is_Deleted] = FALSE()
        )
    )
)
ORDER BY [Subsite Count] DESC
```

### Sites by tenant

```dax
EVALUATE
SUMMARIZE(
    FILTER(Dim_Sites, [Is_Deleted] = FALSE()),
    RELATED(Dim_Tenant[Tenant]),
    "Site Count", COUNTROWS(Dim_Sites),
    "Sites", CONCATENATEX(Dim_Sites, [Site_Name], ", ", [Site_Name], ASC)
)
ORDER BY [Site Count] DESC
```

### Sites with coordinates

```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Dim_Sites,
        "Site", [Site_Name],
        "Latitude", [Site_Latitude],
        "Longitude", [Site_Longitude],
        "Address", [Site_AddressLine1] & ", " & [Site_AddressCity],
        "Is Deleted", [Is_Deleted]
    ),
    NOT ISBLANK([Latitude]) && NOT ISBLANK([Longitude])
)
ORDER BY [Site]
```

---

## Data Model Pattern

**Pattern:** Dimension Table (Legacy Fixed Hierarchy)

**Characteristics:**
- **Legacy Status**: Deprecated in favor of Dim_AssignmentPoints flexible hierarchy
- **Fixed Two-Level Hierarchy**: Site (parent) → Subsite (child)
- **Geographic Data**: Includes address fields and lat/long coordinates for mapping
- **Soft Delete Pattern**: Is_Deleted flag for data retention
- **Multi-Tenant**: Tenant_Id enables tenant isolation

**Historical Context:**
The Site/Subsite hierarchy was the original location structure in Obzervr. It provided a simple two-level hierarchy suitable for organizations with straightforward site structures. However, it was replaced by Dim_AssignmentPoints which offers:
- Flexible depth (up to 9 levels)
- Self-referencing parent-child relationships
- Dynamic hierarchy paths
- Support for complex organizational structures (Area → Location → Point)

**Migration Path:**
Assignments now use Dim_AssignmentPoints[Code] rather than Site_Id/Subsite_Id. Historical assignments may still reference the Site/Subsite tables for backward compatibility.

**Use Cases:**
- Legacy report support
- Historical data analysis where Site/Subsite was used
- Migration validation (mapping old Site/Subsite to new AssignmentPoint hierarchy)
- Geographic reporting for organizations still using the Site/Subsite model

---

## Related Documentation

### ERD Documents
- [ERD #1: Assignment Core Model](../ERDs/ERD_01_Assignment_Core.md) - Primary ERD documenting assignment core tables including legacy Site/Subsite hierarchy

### Related Tables
- **Dim_Subsites** - Child table in Site/Subsite hierarchy (one-to-many relationship)
- **Dim_Tenant** - Tenant configuration table (many-to-one relationship)
- **Dim_AssignmentPoints** - Replacement flexible hierarchy (recommended for new implementations)
- **Assignments** - May reference Site/Subsite in historical records (via calculated columns or legacy columns)

### Other Documentation
- [ERD_Overview.md](../ERD_Overview.md) - Complete semantic model overview with architecture patterns
- [README.md](../README.md) - Documentation maintenance guide and AI regeneration prompts

---

## Change History

| Date | Change Description | Modified By |
|------|-------------------|-------------|
| 2025-11-18 | Initial table documentation created from TMDL metadata | AI Documentation Generator |

---

## Notes

**Legacy Status**: This table is maintained for backward compatibility. New implementations should use Dim_AssignmentPoints for location hierarchies.

**Coordinate Precision**: Latitude and Longitude fields support decimal degree precision suitable for most mapping applications. Consider using spatial data types if advanced geographic operations are required.

**Address Structure**: Address fields follow a flexible structure (AddressLine1-3, City, PostCode) suitable for various international address formats.

**Soft Delete Behavior**: Unlike some other dimension tables, the Power Query source does NOT filter out deleted records. The Is_Deleted column is retained for reporting purposes, allowing reports to show historical data including deleted sites.
