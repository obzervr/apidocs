# Fact_User_Audit_Command_Count_By_Day

## Overview

`Fact_User_Audit_Command_Count_By_Day` is a pre-aggregated fact table that provides daily summaries of user command execution counts across the Obzervr platform. This table serves as a performance-optimized source for user activity analytics, enabling fast queries for user engagement metrics, command usage patterns, and tenant activity tracking without needing to aggregate the raw audit log data at query time.

The table aggregates data by date, user, tenant, and command type, making it ideal for time-series analysis of platform usage, user productivity metrics, and command adoption tracking.

## Columns

| Column Name | Data Type | Description |
|------------|-----------|-------------|
| **Date** | Date | The date of the aggregated command activity. Primary grouping dimension for time-series analysis. |
| **User_Id** | Integer | Foreign key to `RLS_Users`. Identifies the user who executed the commands. |
| **Tenant_Id** | Integer | Foreign key to `Dim_Tenant`. Identifies the tenant context for the commands. |
| **Command** | Text | The name or type of command executed (e.g., "CreateAssignment", "UpdateField", "CompleteWork"). |
| **Command_Count** | Integer | The aggregated count of command executions for this date/user/tenant/command combination. |

**Grain:** One row per unique combination of Date + User_Id + Tenant_Id + Command.

**Key Characteristics:**
- Pre-aggregated daily summaries for performance optimization
- Supports multi-tenant analysis with Tenant_Id dimension
- Enables command-level usage pattern analysis
- Supports both user-level and tenant-level activity metrics

## Relationships

### Active Relationships

- **To `Dim_Calendar`**: Many-to-one relationship on `Date` field
  - Enables time intelligence calculations and date filtering
  - Supports period-over-period comparisons and trending analysis
  
- **To `RLS_Users`**: Many-to-one relationship on `User_Id` field
  - Connects to user dimension for user name, email, team associations
  - Subject to Row-Level Security filtering
  
- **To `Dim_Tenant`**: Many-to-one relationship on `Tenant_Id` field
  - Enables tenant-specific filtering and analysis
  - Supports multi-tenant reporting and tenant comparison metrics

### Inactive Relationships

- None typically configured, as date/user/tenant relationships are usually active

## Data Source

**Type:** Calculated Table (derived from raw audit log data) or SQL View

**Transformation Logic:**
```dax
// Conceptual aggregation pattern (actual implementation may vary)
Fact_User_Audit_Command_Count_By_Day = 
SUMMARIZE(
    Fact_User_Audit_Log,
    Fact_User_Audit_Log[Date],
    Fact_User_Audit_Log[User_Id],
    Fact_User_Audit_Log[Tenant_Id],
    Fact_User_Audit_Log[Command],
    "Command_Count", COUNT(Fact_User_Audit_Log[Audit_Id])
)
```

**Refresh Schedule:**
- Typically refreshed daily to include previous day's activity
- May use incremental refresh to add new dates without full reload
- Historical data remains static once aggregated

## DAX Patterns

### Total Commands by Period
```dax
Total Commands = 
SUM(Fact_User_Audit_Command_Count_By_Day[Command_Count])
```

### Unique Active Users
```dax
Active Users = 
DISTINCTCOUNT(Fact_User_Audit_Command_Count_By_Day[User_Id])
```

### Average Commands Per User Per Day
```dax
Avg Commands Per User = 
DIVIDE(
    SUM(Fact_User_Audit_Command_Count_By_Day[Command_Count]),
    DISTINCTCOUNT(Fact_User_Audit_Command_Count_By_Day[User_Id]),
    0
)
```

### Command Type Distribution
```dax
Command Distribution % = 
VAR CommandTotal = SUM(Fact_User_Audit_Command_Count_By_Day[Command_Count])
VAR TotalCommands = 
    CALCULATE(
        SUM(Fact_User_Audit_Command_Count_By_Day[Command_Count]),
        ALL(Fact_User_Audit_Command_Count_By_Day[Command])
    )
RETURN
    DIVIDE(CommandTotal, TotalCommands, 0)
```

### Top Users by Activity
```dax
User Activity Rank = 
RANKX(
    ALL(RLS_Users[User_Name]),
    CALCULATE(SUM(Fact_User_Audit_Command_Count_By_Day[Command_Count])),
    ,
    DESC,
    Dense
)
```

### Period-Over-Period Growth
```dax
Command Count vs Prior Period = 
VAR CurrentPeriod = SUM(Fact_User_Audit_Command_Count_By_Day[Command_Count])
VAR PriorPeriod = 
    CALCULATE(
        SUM(Fact_User_Audit_Command_Count_By_Day[Command_Count]),
        DATEADD(Dim_Calendar[Date], -1, MONTH)
    )
RETURN
    CurrentPeriod - PriorPeriod
```

### User Engagement Days
```dax
Days Active = 
DISTINCTCOUNT(Fact_User_Audit_Command_Count_By_Day[Date])
```

## Common Usage Scenarios

1. **User Activity Dashboard**
   - Daily active users trend
   - Commands per user metrics
   - User engagement heatmaps

2. **Command Adoption Analysis**
   - Most/least used commands
   - Command usage trends over time
   - Feature adoption metrics

3. **Tenant Usage Monitoring**
   - Tenant activity comparisons
   - Multi-tenant usage distribution
   - Tenant engagement scoring

4. **Productivity Metrics**
   - User productivity rankings
   - Team activity aggregations
   - Period-over-period activity changes

5. **Platform Health Monitoring**
   - Overall platform usage trends
   - User engagement patterns
   - Activity anomaly detection

## Related Tables

- **Dim_Calendar**: Date dimension for time-based analysis
- **RLS_Users**: User details and security context
- **Dim_Tenant**: Tenant information and branding
- **Dim_Teams**: Team associations (via RLS_Users)
- **Fact_User_Audit_Log**: Raw audit log (source for this aggregated table)

## Performance Considerations

- Pre-aggregation significantly improves query performance for daily/weekly/monthly reporting
- Reduces need to scan large raw audit log tables for summary metrics
- Consider partitioning by date for large historical datasets
- Index on User_Id and Tenant_Id columns for optimal filter performance

## Change History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-01-23 | Initial documentation |
