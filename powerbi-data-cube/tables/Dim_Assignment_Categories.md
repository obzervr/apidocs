# Dim_Assignment_Categories

**Table Type:** Dimension Table (Tenant-Specific Lookup)  
**Schema:** dbo.DimAssignmentCategories  
**Primary Key:** Id  
**Related ERD:** [ERD #1: Assignment Core Model](../ERDs/ERD_01_Assignment_Core.md)

---

## Table Overview

Tenant-specific dimension containing assignment category definitions used to classify and organize work orders. Each category includes a business code, display color, and critical flag for priority identification. Categories are configurable per tenant to match organizational work classification schemes.

**Source System:** Analytic Database (dbo.DimAssignmentCategories)

**Row Count:** Varies by tenant (typically 10-50 categories per organization)

**Refresh Type:** Full refresh with tenant filtering

**Multi-Tenant:** Yes (filtered by Tenant_Id)

---

## Table Specifications

| Property | Value |
|----------|-------|
| Lineage Tag | f9bfaa87-6479-4f8b-9e40-c1a8e0f3c92e |
| Query Group | Dimensions |
| Partitions | 1 (Dim_Assignment_Categories) |
| Relationships | 1 outbound to Dim_Tenant, inbound from Assignments (many-to-one) |
| Calculated Columns | 0 |
| Hierarchies | 0 |

---

## Column Specifications

| Column Name | Data Type | Description | Source Column | Key | Nullable |
|-------------|-----------|-------------|---------------|-----|----------|
| **Tenant_Id** | string | Tenant identifier | TenantId | FK | No |
| **Id** | string | Unique category identifier | Id | PK | No |
| **Category** | string | Category display name | Category | - | Yes |
| **Code** | string | Business code for category | Code | - | Yes |
| **Color** | string | Hex color code for visualization | Color | - | Yes |
| **Critical** | boolean | Flag indicating critical/high priority category | Critical | - | Yes |
| **Last_Updated** | datetime | Last update timestamp | LastUpdated | - | Yes |
| **Created_Date** | datetime | Creation timestamp | CreatedDate | - | Yes |
| **LastLoaded** | datetime | ETL load timestamp | LastLoaded | - | Yes |

**Note:** IsDeleted column filtered out during Power Query transformation (only active categories loaded)

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
- **Purpose:** Links categories to tenant configuration

### Inbound Relationships

**From Assignments**
- **Type:** Many-to-one
- **From Column:** Assignments[Category_Id]
- **To Column:** Dim_Assignment_Categories[Id]
- **Cardinality:** Many:1
- **Cross-filter Direction:** Single
- **Status:** Active
- **Purpose:** Links assignments to their category classification

---

## Power Query M Source

```m
let
    Source = Sql.Database(#"SQL Server", #"Analytic Database", [CreateNavigationProperties=false, MultiSubnetFailover=true]),
    Table = Source{[Schema="dbo",Item="DimAssignmentCategories"]}[Data],
    #"Filtered Rows" = Table.SelectRows(Table, each AllTenants = true or [TenantId] = #"TenantId1" or [TenantId] = TenantId2 or [TenantId] = TenantId3 or [TenantId] = TenantId4 or [TenantId] = TenantId5),
    #"Removed Columns" = Table.RemoveColumns(#"Filtered Rows",{"IsDeleted"}),
    #"Renamed Columns" = Table.RenameColumns(#"Removed Columns",{
        {"TenantId", "Tenant_Id"}, 
        {"CreatedDate", "Created_Date"}, 
        {"LastUpdated", "Last_Updated"}
    })
in
    #"Renamed Columns"
```

**Key Transformations:**
1. SQL database connection with MultiSubnetFailover
2. Tenant filtering using parameters
3. Removed IsDeleted column (soft delete filtering in Power Query)
4. Column renaming to snake_case

**Filtering Behavior:** Unlike Dim_Sites and Dim_Subsites which retain the Is_Deleted column, this table removes deleted categories entirely in Power Query, ensuring only active categories are available in the model.

---

## DAX Query Patterns

### Category list with assignment counts

```dax
EVALUATE
ADDCOLUMNS(
    Dim_Assignment_Categories,
    "Category", [Category],
    "Code", [Code],
    "Color", [Color],
    "Is Critical", [Critical],
    "Tenant", RELATED(Dim_Tenant[Tenant]),
    "Assignment Count", COUNTROWS(RELATEDTABLE(Assignments))
)
ORDER BY [Tenant], [Category]
```

### Critical categories only

```dax
EVALUATE
FILTER(
    SELECTCOLUMNS(
        Dim_Assignment_Categories,
        "Category", [Category],
        "Code", [Code],
        "Color", [Color],
        "Tenant", RELATED(Dim_Tenant[Tenant])
    ),
    [Critical] = TRUE()
)
ORDER BY [Tenant], [Category]
```

### Category distribution by tenant

```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_Tenant[Tenant],
    Dim_Assignment_Categories[Category],
    Dim_Assignment_Categories[Critical],
    "Total Assignments", COUNTROWS(Assignments),
    "Critical Flag", IF(Dim_Assignment_Categories[Critical], "Critical", "Standard")
)
ORDER BY [Tenant], [Total Assignments] DESC
```

### Category color measure for conditional formatting

```dax
Category Color = 
SELECTEDVALUE(Dim_Assignment_Categories[Color], "#CCCCCC")

// Use in table or matrix visuals with conditional formatting
// to display category with matching background color
```

### Critical assignment count

```dax
Critical Assignments = 
CALCULATE(
    COUNTROWS(Assignments),
    Dim_Assignment_Categories[Critical] = TRUE()
)
```

---

## Data Model Pattern

**Pattern:** Tenant-Specific Lookup Dimension with Visual Styling

**Characteristics:**
- **Tenant Configuration**: Each tenant defines their own category structure
- **Business Codes**: Code column enables integration with external systems
- **Visual Styling**: Color column provides hex codes for consistent visualization
- **Priority Flagging**: Critical boolean enables priority identification and filtering
- **Active Records Only**: Power Query removes soft-deleted categories (no Is_Deleted column in model)

**Common Category Types:**
- **Preventive Maintenance**: Scheduled routine maintenance
- **Corrective Maintenance**: Repair work
- **Inspection**: Assessment and audit activities
- **Emergency**: Urgent/critical repairs
- **Compliance**: Regulatory and safety requirements
- **Project**: Capital projects and upgrades

**Critical Flag Usage:**
The Critical column enables:
1. **Priority Filtering**: Show only critical assignments
2. **Alert Triggers**: Highlight critical work in dashboards
3. **Resource Allocation**: Prioritize critical categories in scheduling
4. **KPI Tracking**: Separate metrics for critical vs. standard work

**Color Coding:**
Categories are color-coded for visual consistency:
- Color hex codes stored in Color column
- Use SELECTEDVALUE in DAX measures for conditional formatting
- Enables consistent visualization across all reports without hardcoded colors

---

## Related Documentation

### ERD Documents
- [ERD #1: Assignment Core Model](../ERDs/ERD_01_Assignment_Core.md) - Documents assignment classification

### Related Tables
- **Assignments** - Main fact table using Category_Id FK
- **Dim_Tenant** - Tenant configuration
- **Dim_Assignment_Status** - Related assignment classification dimension

### Other Documentation
- [ERD_Overview.md](../ERD_Overview.md) - Architecture patterns including tenant-specific dimensions
- [Assignments.md](Assignments.md) - Assignments fact table with category relationships

---

## Change History

| Date | Change Description | Modified By |
|------|-------------------|-------------|
| 2025-11-18 | Initial table documentation created from TMDL metadata | AI Documentation Generator |

---

## Notes

**Tenant Customization**: Each tenant configures their own category structure to match organizational work classification schemes. Category names, codes, and critical flags are tenant-specific.

**Soft Delete Behavior**: Unlike Dim_Sites and Dim_Subsites which retain Is_Deleted columns in the model, this table filters out deleted categories in Power Query. Only active categories are available for assignment classification.

**Color Consistency**: The Color column ensures consistent color-coding across all reports. Categories maintain visual identity across different report pages and dashboards.

**Critical Flag Strategy**: The Critical boolean enables priority-based filtering and reporting without requiring separate category hierarchies or classification schemes. Critical assignments can be tracked independently of category names.

**Code Column Purpose**: The Code column provides stable business identifiers for categories, enabling:
- Integration with external systems (WMS, ERP)
- Category mapping during data migrations
- Consistent reference across multiple semantic models
- API integration for mobile applications

**Typical Critical Categories**: Emergency repairs, safety hazards, regulatory compliance issues, production downtime events
