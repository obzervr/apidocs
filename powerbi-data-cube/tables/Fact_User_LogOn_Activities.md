# Fact_User_LogOn_Activities

## Table Overview
`Fact_User_LogOn_Activities` is a fact table that tracks user authentication events within the Obzervr system. Each row represents a single user logon activity, enabling usage tracking, access auditing, and user engagement analysis.

This table provides simple activity tracking with timestamp and date key for temporal analysis.

**Current Status**: Standard import table (no incremental refresh specified).

---

## Specifications
- **Source**: `FactCosmosUserActivitySnapshot` table
- **Row Count**: High volume (one row per logon event)
- **Grain**: One row per user logon activity
- **Primary Key**: Composite (User_Id + Activity_Timestamp)
- **Incremental Refresh**: Not enabled
- **Partitioning Strategy**: Standard import
- **Source Columns**: 2
- **Calculated Columns**: 1 (Date_Key)

---

## Column Specifications

| Column Name | Data Type | Format | Nullable | Hidden | Description |
|------------|-----------|--------|----------|--------|-------------|
| User_Id | Int64 | | No | No | Foreign key to user dimension |
| Activity_Timestamp | DateTime | yyyy-MM-dd HH:mm:ss | No | No | Timestamp when user logged on |
| Date_Key | Int64 | YYYYMMDD | No | No | Date key derived from Activity_Timestamp for date dimension relationship |

---

## Calculated Columns

### Date_Key
Calculated date key in YYYYMMDD format derived from the `Activity_Timestamp` for efficient relationship to the date dimension.

```dax
Date_Key = 
YEAR(Fact_User_LogOn_Activities[Activity_Timestamp]) * 10000
+ MONTH(Fact_User_LogOn_Activities[Activity_Timestamp]) * 100
+ DAY(Fact_User_LogOn_Activities[Activity_Timestamp])
```

---

## Relationships

### Outbound Relationships
| To Table | From Column(s) | To Column(s) | Cardinality | Cross Filter | Relationship ID |
|----------|---------------|--------------|-------------|--------------|-----------------|
| `Dim_User_Reference` | User_Id | User_Id | Many-to-One | Single | (User context) |
| `Dim_Date_Reference` | Date_Key | Date_Key | Many-to-One | Single | (Date context) |

### Inbound Relationships
None. This is a leaf table in the relationship hierarchy.

---

## Power Query M Source

```m
let
    Source = Value.NativeQuery(
        Obzervr_DataWarehouse,
        "SELECT 
            User_Id,
            Activity_Timestamp
        FROM [dbo].[FactCosmosUserActivitySnapshot]"
    ),
    #"Removed Snapshot_Timestamp Column" = Table.RemoveColumns(
        Source,
        {"Snapshot_Timestamp"}
    ),
    #"Filtered Tenant" = Table.SelectRows(
        #"Removed Snapshot_Timestamp Column",
        each [TenantId] = TenantId
    ),
    #"Removed TenantId Column" = Table.RemoveColumns(
        #"Filtered Tenant",
        {"TenantId"}
    ),
    #"Changed Types" = Table.TransformColumnTypes(
        #"Removed TenantId Column",
        {
            {"User_Id", Int64.Type},
            {"Activity_Timestamp", type datetime}
        }
    )
in
    #"Changed Types"
```

---

## DAX Query Patterns

### Example 1: Daily Logon Trend
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_Date_Reference[Full_Date],
    "Logon_Count", COUNTROWS(Fact_User_LogOn_Activities),
    "Unique_Users", DISTINCTCOUNT(Fact_User_LogOn_Activities[User_Id]),
    "Avg_Logons_Per_User", DIVIDE(
        COUNTROWS(Fact_User_LogOn_Activities),
        DISTINCTCOUNT(Fact_User_LogOn_Activities[User_Id]),
        0
    )
)
ORDER BY Dim_Date_Reference[Full_Date] DESC
```

### Example 2: Most Active Users
```dax
EVALUATE
TOPN(
    50,
    ADDCOLUMNS(
        SUMMARIZE(Fact_User_LogOn_Activities, Fact_User_LogOn_Activities[User_Id]),
        "User_Name", RELATED(Dim_User_Reference[Full_Name]),
        "Total_Logons", COUNTROWS(Fact_User_LogOn_Activities),
        "First_Logon", MIN(Fact_User_LogOn_Activities[Activity_Timestamp]),
        "Latest_Logon", MAX(Fact_User_LogOn_Activities[Activity_Timestamp]),
        "Days_Active", DATEDIFF(
            MIN(Fact_User_LogOn_Activities[Activity_Timestamp]),
            MAX(Fact_User_LogOn_Activities[Activity_Timestamp]),
            DAY
        ) + 1,
        "Avg_Daily_Logons", DIVIDE(
            COUNTROWS(Fact_User_LogOn_Activities),
            DATEDIFF(
                MIN(Fact_User_LogOn_Activities[Activity_Timestamp]),
                MAX(Fact_User_LogOn_Activities[Activity_Timestamp]),
                DAY
            ) + 1,
            0
        )
    ),
    [Total_Logons],
    DESC
)
```

### Example 3: Logon Activity by Hour of Day
```dax
EVALUATE
ADDCOLUMNS(
    SUMMARIZE(
        Fact_User_LogOn_Activities,
        "Hour", HOUR(Fact_User_LogOn_Activities[Activity_Timestamp])
    ),
    "Logon_Count", COUNTROWS(Fact_User_LogOn_Activities),
    "Unique_Users", DISTINCTCOUNT(Fact_User_LogOn_Activities[User_Id]),
    "Avg_Logons_Per_User", DIVIDE(
        COUNTROWS(Fact_User_LogOn_Activities),
        DISTINCTCOUNT(Fact_User_LogOn_Activities[User_Id]),
        0
    )
)
ORDER BY [Hour]
```

### Example 4: User Engagement Trend
```dax
EVALUATE
SUMMARIZECOLUMNS(
    Dim_Date_Reference[Calendar_Year],
    Dim_Date_Reference[Calendar_Month_Name],
    "Active_Users", DISTINCTCOUNT(Fact_User_LogOn_Activities[User_Id]),
    "Total_Logons", COUNTROWS(Fact_User_LogOn_Activities),
    "Avg_Logons_Per_User_Per_Month", DIVIDE(
        COUNTROWS(Fact_User_LogOn_Activities),
        DISTINCTCOUNT(Fact_User_LogOn_Activities[User_Id]),
        0
    )
)
ORDER BY Dim_Date_Reference[Calendar_Year], Dim_Date_Reference[Month_Number]
```

### Example 5: Inactive Users Report
```dax
EVALUATE
VAR LastLogonByUser = 
    SUMMARIZE(
        Fact_User_LogOn_Activities,
        Fact_User_LogOn_Activities[User_Id],
        "Last_Logon", MAX(Fact_User_LogOn_Activities[Activity_Timestamp])
    )
RETURN
    ADDCOLUMNS(
        FILTER(
            LastLogonByUser,
            DATEDIFF([Last_Logon], TODAY(), DAY) > 30
        ),
        "User_Name", LOOKUPVALUE(
            Dim_User_Reference[Full_Name],
            Dim_User_Reference[User_Id], [User_Id]
        ),
        "Days_Since_Last_Logon", DATEDIFF([Last_Logon], TODAY(), DAY),
        "Total_Historical_Logons", CALCULATE(
            COUNTROWS(Fact_User_LogOn_Activities),
            FILTER(
                ALL(Fact_User_LogOn_Activities),
                Fact_User_LogOn_Activities[User_Id] = EARLIER([User_Id])
            )
        )
    )
ORDER BY [Days_Since_Last_Logon] DESC
```

### Example 6: Weekend vs Weekday Logon Analysis
```dax
EVALUATE
ADDCOLUMNS(
    SUMMARIZE(
        Fact_User_LogOn_Activities,
        "Is_Weekend", IF(
            WEEKDAY(Fact_User_LogOn_Activities[Activity_Timestamp], 2) >= 6,
            "Weekend",
            "Weekday"
        )
    ),
    "Logon_Count", COUNTROWS(Fact_User_LogOn_Activities),
    "Unique_Users", DISTINCTCOUNT(Fact_User_LogOn_Activities[User_Id]),
    "Pct_Of_Total", DIVIDE(
        COUNTROWS(Fact_User_LogOn_Activities),
        CALCULATE(COUNTROWS(Fact_User_LogOn_Activities), ALL(Fact_User_LogOn_Activities)),
        0
    )
)
```

---

## Data Model Pattern

### Simple Activity Log Pattern
`Fact_User_LogOn_Activities` implements a simple activity logging pattern that captures authentication events with minimal attributes. This lightweight pattern enables usage tracking and engagement analysis without complex event hierarchies.

**Activity Tracking Characteristics**:
- **Single Event Type**: Only logon events (no logoff, session duration, or activity types)
- **Timestamp Precision**: Full datetime precision for time-of-day analysis
- **User Context**: Single foreign key to user dimension
- **Date Dimension Link**: Calculated Date_Key enables temporal analysis

**Minimal Fact Pattern**:
Unlike complex fact tables with many measures and dimensions, this table captures only:
- Who (User_Id)
- When (Activity_Timestamp)

This simplicity provides:
- Fast loading and refresh
- Efficient storage
- Simple query patterns
- Easy aggregation

**Date_Key Calculation**:
The DAX-calculated Date_Key (YYYYMMDD format) enables:
- Relationship to Dim_Date_Reference for calendar intelligence
- Aggregation by date periods (day, week, month, year)
- Relative date filtering (last 30 days, YTD, etc.)

**Usage Patterns**:
This table supports common usage analytics:
1. **Active User Counts**: DISTINCTCOUNT(User_Id) by period
2. **Logon Frequency**: COUNTROWS per user per period
3. **Time-of-Day Analysis**: HOUR(Activity_Timestamp) aggregation
4. **Engagement Trends**: Tracking active users over time
5. **Inactive User Detection**: Users without recent logons

**Snapshot_Timestamp Removal**:
The Power Query removes a `Snapshot_Timestamp` column from the source, suggesting:
- Source table may capture snapshots at intervals
- Activity_Timestamp is the relevant event time
- Snapshot timing is not needed for reporting

**No Incremental Refresh**:
The absence of incremental refresh suggests either:
- Data volume is manageable for full refresh
- Source table is already a recent snapshot/subset
- Logon history may be truncated or archived at source

**Example Scenario - User Engagement Tracking**:

**Daily Logon Summary** (2024-11-15):
| Hour | Logons | Unique_Users | Peak_Activity |
|------|--------|--------------|---------------|
| 06:00 | 12 | 12 | Day shift start |
| 07:00 | 45 | 42 | Morning peak |
| 08:00 | 28 | 18 | Multi-device logons |
| 12:00 | 15 | 15 | Lunch shift change |
| 14:00 | 32 | 28 | Swing shift start |
| 18:00 | 8 | 8 | Evening activity |
| 22:00 | 18 | 16 | Night shift start |

**User Activity Profile** - John Smith (User_Id 42):
- First Logon: 2024-01-10 07:15:23
- Latest Logon: 2024-11-15 06:58:12
- Total Logons: 487
- Days Active: 280 days
- Avg Daily Logons: 1.74
- Typical Hours: 07:00, 14:30 (day shift pattern)

**Engagement Trends** (Last 12 Months):
| Month | Active Users | Total Logons | Avg Logons/User |
|-------|-------------|--------------|-----------------|
| Nov 2023 | 125 | 2,850 | 22.8 |
| Dec 2023 | 118 | 2,420 | 20.5 |
| Jan 2024 | 132 | 3,100 | 23.5 |
| ... | ... | ... | ... |
| Oct 2024 | 145 | 3,680 | 25.4 |
| Nov 2024 | 148 | 3,920 | 26.5 |

Analysis insights:
- User base growing (125 → 148 active users)
- Engagement increasing (22.8 → 26.5 avg logons/user)
- Peak hours align with shift changes
- Weekend activity minimal (5-10% of weekday volume)

---

## Related Documentation
- **ERD_07_Audit_Activity.md** - ERD diagram showing activity and audit relationship context
- **Dim_User_Reference.md** - User dimension for activity context
- **Dim_Date_Reference.md** - Date dimension for temporal analysis
- **Fact_User_Assignment_Audit.md** - User command audit trail (action-level tracking)

---

## Change History
| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-15 | Auto-generated | Initial documentation from TMDL metadata |

---

## Notes
- **Simple Activity Log**: This table captures only logon events with minimal attributes (user, timestamp), providing lightweight usage tracking.
- **No Logoff Events**: The table does not capture logoff events or session duration, only authentication timestamps.
- **Date_Key Calculation**: Calculated in DAX rather than Power Query, using YYYYMMDD formula for efficient date dimension relationship.
- **Snapshot_Timestamp Removed**: The source column `Snapshot_Timestamp` is removed in Power Query, retaining only the activity event timestamp.
- **No Incremental Refresh**: Unlike many fact tables, this uses standard import without incremental refresh, suggesting manageable data volume or source-level data retention.
- **Composite Primary Key**: While not explicitly enforced, the combination of User_Id + Activity_Timestamp serves as the natural key (users can have multiple logons, each with unique timestamp).
- **Minimal Dimensions**: Only two dimensional relationships (user and date), keeping the model simple and focused.
- **Time-of-Day Analysis**: Full datetime precision enables hour-of-day, day-of-week, and shift-based analysis patterns.
- **Engagement Metrics**: Primary use case is tracking active users, logon frequency, and engagement trends over time.
- **No Activity Types**: Unlike more complex activity tables, this captures only one event type (logon), not various user actions.
- **Tenant Filtering**: Standard tenant filtering applied in Power Query, though TenantId column is removed after filtering.
- **Source Table Name**: `FactCosmosUserActivitySnapshot` suggests Cosmos DB source with periodic snapshots, though snapshot timestamp is not retained.
- **Usage Reporting**: Ideal for executive dashboards showing user adoption, active users, and engagement trends.
- **Inactive User Detection**: Absence of recent records enables identification of inactive users requiring follow-up or license reclamation.
- **Multiple Logons**: Users may have multiple logon records per day (multi-device, session timeouts, application restarts).
- **No Session ID**: The table doesn't include session identifiers, so individual session duration cannot be calculated.
- **Timezone Considerations**: Activity_Timestamp timezone handling should be considered when analyzing time-of-day patterns across geographic regions.
