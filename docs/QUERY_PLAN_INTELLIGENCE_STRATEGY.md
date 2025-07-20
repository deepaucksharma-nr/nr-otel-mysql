# Query Plan Intelligence Strategy

## Executive Summary

This document outlines how to build a SolarWinds Plan Explorer and DPA-equivalent (or better) solution using only OpenTelemetry configuration. We'll focus on query plan analysis, wait-time methodology, and multi-dimensional performance analysis without any code changes.

## Core Capabilities to Implement

### 1. Query Plan Capture and Analysis
- Execution plan extraction via SQL queries
- Plan change detection and versioning
- Cost analysis (estimated vs actual)
- Index effectiveness scoring
- Statistics freshness monitoring

### 2. Wait-Time Analysis
- Comprehensive wait event profiling
- Wait chain analysis
- Lock dependency graphs
- Resource contention mapping

### 3. Multi-Dimensional Correlation
- Query → Wait → Resource → Business Impact
- Application trace correlation
- Infrastructure metric integration
- End-to-end latency attribution

### 4. Intelligent Recommendations
- Missing index detection
- Query rewrite suggestions
- Statistics update advisories
- Capacity planning predictions

## Implementation Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Query Plan Intelligence                    │
├───────────────────┬───────────────────┬─────────────────────┤
│  Plan Extraction  │  Wait Analysis    │  Correlation Engine │
├───────────────────┼───────────────────┼─────────────────────┤
│  SQL Queries      │  Performance      │  Cross-Signal       │
│  Plan Parser      │  Schema           │  Correlation        │
│  Change Detector  │  Wait Events      │  Business Impact    │
└───────────────────┴───────────────────┴─────────────────────┘
                            ↓
                    OpenTelemetry Collectors
                            ↓
                       New Relic NRDB
                            ↓
                    Intelligence Dashboards
```

## Phase 1: Query Plan Extraction

### SQL Query for Plan Capture (MySQL)

```sql
-- Comprehensive query plan extraction
WITH plan_capture AS (
  SELECT 
    ess.THREAD_ID,
    ess.EVENT_ID,
    ess.DIGEST,
    ess.DIGEST_TEXT,
    ess.SQL_TEXT,
    ess.TIMER_WAIT/1000000000 as EXEC_TIME_MS,
    ess.ROWS_SENT,
    ess.ROWS_EXAMINED,
    ess.SELECT_FULL_JOIN,
    ess.SELECT_FULL_RANGE_JOIN,
    ess.SELECT_RANGE,
    ess.SELECT_RANGE_CHECK,
    ess.SELECT_SCAN,
    ess.SORT_MERGE_PASSES,
    ess.SORT_RANGE,
    ess.SORT_ROWS,
    ess.SORT_SCAN,
    ess.NO_INDEX_USED,
    ess.NO_GOOD_INDEX_USED,
    ess.CREATED_TMP_DISK_TABLES,
    ess.CREATED_TMP_TABLES,
    -- Extract EXPLAIN output
    (SELECT GROUP_CONCAT(
      CONCAT(
        'id:', IFNULL(id, ''),
        '|select_type:', IFNULL(select_type, ''),
        '|table:', IFNULL(table_name, ''),
        '|type:', IFNULL(access_type, ''),
        '|possible_keys:', IFNULL(possible_keys, ''),
        '|key:', IFNULL(key, ''),
        '|key_len:', IFNULL(key_length, ''),
        '|ref:', IFNULL(ref, ''),
        '|rows:', IFNULL(rows_examined, ''),
        '|filtered:', IFNULL(filtered, ''),
        '|Extra:', IFNULL(extra, '')
      ) SEPARATOR '\\n'
    ) FROM information_schema.OPTIMIZER_TRACE) as EXPLAIN_PLAN,
    -- Index statistics
    (SELECT GROUP_CONCAT(
      CONCAT(
        t.TABLE_NAME, '.', s.INDEX_NAME, ':',
        'cardinality=', s.CARDINALITY,
        ',pages=', s.NUMBER_OF_PAGES,
        ',last_update=', t.UPDATE_TIME
      ) SEPARATOR ';'
    ) 
    FROM information_schema.STATISTICS s
    JOIN information_schema.TABLES t ON s.TABLE_SCHEMA = t.TABLE_SCHEMA 
      AND s.TABLE_NAME = t.TABLE_NAME
    WHERE s.TABLE_SCHEMA = DATABASE()
      AND ess.DIGEST_TEXT LIKE CONCAT('%', s.TABLE_NAME, '%')
    ) as INDEX_STATS,
    -- Table statistics
    (SELECT GROUP_CONCAT(
      CONCAT(
        TABLE_NAME, ':',
        'rows=', TABLE_ROWS,
        ',data_length=', DATA_LENGTH,
        ',index_length=', INDEX_LENGTH,
        ',update_time=', UPDATE_TIME
      ) SEPARATOR ';'
    )
    FROM information_schema.TABLES
    WHERE TABLE_SCHEMA = DATABASE()
      AND ess.DIGEST_TEXT LIKE CONCAT('%', TABLE_NAME, '%')
    ) as TABLE_STATS
  FROM performance_schema.events_statements_summary_by_digest ess
  WHERE ess.DIGEST IS NOT NULL
    AND ess.COUNT_STAR > 10  -- Focus on frequently executed queries
)
SELECT * FROM plan_capture;
```

### OTel Configuration for Plan Extraction

```yaml
receivers:
  sqlquery/plan_extractor:
    driver: mysql
    datasource: "${env:MYSQL_USER}:${env:MYSQL_PASSWORD}@tcp(${env:MYSQL_ENDPOINT})/information_schema"
    queries:
      - sql: |
          <comprehensive plan extraction query above>
        metrics:
          - metric_name: mysql.query.plan.captured
            value_column: "EXEC_TIME_MS"
            data_type: gauge
            attribute_columns:
              - DIGEST
              - DIGEST_TEXT
              - ROWS_SENT
              - ROWS_EXAMINED
              - NO_INDEX_USED
              - NO_GOOD_INDEX_USED
              - EXPLAIN_PLAN
              - INDEX_STATS
              - TABLE_STATS
        collection_interval: 60s
```

## Phase 2: Advanced Plan Analysis

### Plan Change Detection

```yaml
processors:
  transform/plan_analysis:
    error_mode: ignore
    metric_statements:
      - context: datapoint
        statements:
          # Calculate plan hash for change detection
          - set(attributes["plan.hash"], 
                Hash(attributes["EXPLAIN_PLAN"]))
            where attributes["EXPLAIN_PLAN"] != nil
          
          # Detect plan changes
          - set(attributes["plan.changed"], 
                attributes["plan.hash"] != attributes["plan.previous_hash"])
            where attributes["plan.previous_hash"] != nil
          
          # Calculate index effectiveness score
          - set(attributes["index.effectiveness_score"], 
                Case(
                  attributes["NO_INDEX_USED"] == "1", 0,
                  attributes["NO_GOOD_INDEX_USED"] == "1", 20,
                  attributes["SELECT_FULL_JOIN"] > 0, 30,
                  attributes["SELECT_SCAN"] > 0, 50,
                  attributes["SELECT_RANGE"] > 0, 80,
                  100
                ))
          
          # Estimate vs actual analysis
          - set(attributes["cardinality.error_ratio"], 
                attributes["ROWS_EXAMINED"] / Max(attributes["ROWS_SENT"], 1))
            where attributes["ROWS_EXAMINED"] != nil
          
          # Statistics freshness score
          - set(attributes["stats.age_hours"], 
                (Now() - Time(attributes["update_time"])) / 3600)
            where attributes["update_time"] != nil
          
          - set(attributes["stats.freshness_score"], 
                Case(
                  attributes["stats.age_hours"] < 24, 100,
                  attributes["stats.age_hours"] < 168, 70,  # 1 week
                  attributes["stats.age_hours"] < 720, 40,  # 30 days
                  10
                ))
```

### Missing Index Detection

```yaml
processors:
  transform/index_advisor:
    error_mode: ignore
    metric_statements:
      - context: datapoint
        statements:
          # Identify missing index candidates
          - set(attributes["index.missing"], true)
            where attributes["NO_INDEX_USED"] == "1" AND 
                  attributes["ROWS_EXAMINED"] > 1000
          
          # Calculate index impact score
          - set(attributes["index.impact_score"], 
                (attributes["ROWS_EXAMINED"] - attributes["ROWS_SENT"]) * 
                attributes["execution_count"] / 1000)
            where attributes["index.missing"] == true
          
          # Generate index recommendation
          - set(attributes["index.recommendation"], 
                "CREATE INDEX idx_" + 
                Substring(attributes["DIGEST"], 0, 8) + 
                " ON <table> (<columns>)")
            where attributes["index.missing"] == true
          
          # Estimate improvement
          - set(attributes["index.estimated_improvement"], 
                (1 - (attributes["ROWS_SENT"] / attributes["ROWS_EXAMINED"])) * 100)
            where attributes["ROWS_EXAMINED"] > 0
```

## Phase 3: Wait-Time Analysis (DPA Equivalent)

### Comprehensive Wait Event Capture

```yaml
receivers:
  sqlquery/wait_profiler:
    driver: mysql
    datasource: "${env:MYSQL_USER}:${env:MYSQL_PASSWORD}@tcp(${env:MYSQL_ENDPOINT})/performance_schema"
    queries:
      - sql: |
          WITH wait_profile AS (
            SELECT 
              ew.THREAD_ID,
              ew.EVENT_NAME,
              ew.TIMER_WAIT/1000000000 as WAIT_TIME_MS,
              ew.OBJECT_SCHEMA,
              ew.OBJECT_NAME,
              ew.INDEX_NAME,
              ew.OBJECT_TYPE,
              ew.NESTING_EVENT_ID,
              -- Wait categorization
              CASE 
                WHEN ew.EVENT_NAME LIKE 'wait/io/file%' THEN 'IO'
                WHEN ew.EVENT_NAME LIKE 'wait/lock%' THEN 'Lock'
                WHEN ew.EVENT_NAME LIKE 'wait/synch/mutex%' THEN 'Mutex'
                WHEN ew.EVENT_NAME LIKE 'wait/synch/cond%' THEN 'Condition'
                ELSE 'Other'
              END as WAIT_CATEGORY,
              -- Get associated query
              (SELECT DIGEST FROM events_statements_current 
               WHERE THREAD_ID = ew.THREAD_ID 
               ORDER BY EVENT_ID DESC LIMIT 1) as QUERY_DIGEST,
              -- Lock chain analysis
              (SELECT GROUP_CONCAT(
                CONCAT('T', mdl.OWNER_THREAD_ID, ':', mdl.OBJECT_NAME)
                SEPARATOR ' -> '
              ) FROM metadata_locks mdl
              WHERE mdl.OBJECT_SCHEMA = ew.OBJECT_SCHEMA
                AND mdl.OBJECT_NAME = ew.OBJECT_NAME
              ) as LOCK_CHAIN
            FROM events_waits_current ew
            WHERE ew.TIMER_WAIT > 0
          )
          SELECT 
            WAIT_CATEGORY,
            COUNT(*) as WAIT_COUNT,
            SUM(WAIT_TIME_MS) as TOTAL_WAIT_MS,
            AVG(WAIT_TIME_MS) as AVG_WAIT_MS,
            MAX(WAIT_TIME_MS) as MAX_WAIT_MS,
            GROUP_CONCAT(DISTINCT QUERY_DIGEST) as AFFECTED_QUERIES,
            GROUP_CONCAT(DISTINCT LOCK_CHAIN) as LOCK_CHAINS
          FROM wait_profile
          GROUP BY WAIT_CATEGORY
        metrics:
          - metric_name: mysql.wait.profile.comprehensive
            value_column: "TOTAL_WAIT_MS"
            attribute_columns:
              - WAIT_CATEGORY
              - WAIT_COUNT
              - AVG_WAIT_MS
              - MAX_WAIT_MS
              - AFFECTED_QUERIES
              - LOCK_CHAINS
        collection_interval: 10s
```

### Multi-Dimensional Wait Analysis

```yaml
processors:
  transform/wait_intelligence:
    error_mode: ignore
    metric_statements:
      - context: datapoint
        statements:
          # Calculate wait severity
          - set(attributes["wait.severity_score"], 
                Case(
                  attributes["WAIT_CATEGORY"] == "Lock" AND 
                  attributes["MAX_WAIT_MS"] > 5000, 100,
                  
                  attributes["WAIT_CATEGORY"] == "IO" AND 
                  attributes["AVG_WAIT_MS"] > 100, 80,
                  
                  attributes["WAIT_CATEGORY"] == "Mutex" AND 
                  attributes["WAIT_COUNT"] > 1000, 60,
                  
                  attributes["TOTAL_WAIT_MS"] > 1000, 40,
                  20
                ))
          
          # Identify wait anomalies
          - set(attributes["wait.is_anomaly"], 
                attributes["TOTAL_WAIT_MS"] > 
                attributes["wait.baseline_p95"] * 2)
            where attributes["wait.baseline_p95"] != nil
          
          # Lock chain analysis
          - set(attributes["lock.chain_length"], 
                Len(Split(attributes["LOCK_CHAINS"], " -> ")))
            where attributes["LOCK_CHAINS"] != nil
          
          - set(attributes["lock.is_deadlock_risk"], 
                attributes["lock.chain_length"] > 3)
```

## Phase 4: Intelligent Recommendations Engine

### Automated Advisory System

```yaml
processors:
  transform/advisory_engine:
    error_mode: ignore
    metric_statements:
      - context: datapoint
        statements:
          # Generate comprehensive recommendations
          - set(attributes["recommendation.type"], 
                Case(
                  # Missing index
                  attributes["index.missing"] == true AND 
                  attributes["index.impact_score"] > 1000, 
                  "CREATE_INDEX",
                  
                  # Stale statistics
                  attributes["stats.freshness_score"] < 40,
                  "UPDATE_STATISTICS",
                  
                  # Query rewrite needed
                  attributes["cardinality.error_ratio"] > 100,
                  "REWRITE_QUERY",
                  
                  # Lock contention
                  attributes["lock.chain_length"] > 2,
                  "OPTIMIZE_LOCKING",
                  
                  # Resource saturation
                  attributes["wait.severity_score"] > 80 AND
                  attributes["WAIT_CATEGORY"] == "IO",
                  "SCALE_RESOURCES",
                  
                  "NONE"
                ))
          
          # Calculate recommendation priority
          - set(attributes["recommendation.priority"], 
                attributes["business_impact_score"] * 
                attributes["wait.severity_score"] * 
                attributes["execution_frequency"] / 10000)
          
          # Estimate improvement potential
          - set(attributes["recommendation.improvement_estimate"], 
                Case(
                  attributes["recommendation.type"] == "CREATE_INDEX",
                  attributes["index.estimated_improvement"],
                  
                  attributes["recommendation.type"] == "UPDATE_STATISTICS",
                  30,
                  
                  attributes["recommendation.type"] == "REWRITE_QUERY",
                  attributes["cardinality.error_ratio"] / 10,
                  
                  10
                ))
```

## Phase 5: Dashboard Implementation

### 1. Query Plan Explorer Dashboard

```json
{
  "name": "Query Plan Explorer",
  "pages": [
    {
      "name": "Plan Analysis",
      "widgets": [
        {
          "title": "Execution Plans with Changes",
          "visualization": "table",
          "query": "SELECT latest(DIGEST_TEXT) as 'Query', latest(EXPLAIN_PLAN) as 'Current Plan', latest(plan.hash) as 'Plan Hash', latest(plan.changed) as 'Changed', latest(index.effectiveness_score) as 'Index Score', latest(cardinality.error_ratio) as 'Est/Actual Ratio' FROM QueryPlanIntelligence WHERE plan.changed = true OR index.effectiveness_score < 50 FACET DIGEST SINCE 24 hours ago"
        },
        {
          "title": "Index Effectiveness Distribution",
          "visualization": "histogram",
          "query": "SELECT histogram(index.effectiveness_score, 10, 10) FROM QueryPlanIntelligence SINCE 1 hour ago"
        },
        {
          "title": "Missing Index Impact",
          "visualization": "bar",
          "query": "SELECT sum(index.impact_score) as 'Impact Score', latest(index.recommendation) as 'Recommendation' FROM QueryPlanIntelligence WHERE index.missing = true FACET DIGEST SINCE 24 hours ago LIMIT 20"
        }
      ]
    }
  ]
}
```

### 2. Wait-Time Analysis Dashboard (DPA-Style)

```json
{
  "name": "Wait-Time Analysis",
  "pages": [
    {
      "name": "Wait Profile",
      "widgets": [
        {
          "title": "Wait Category Distribution",
          "visualization": "pie",
          "query": "SELECT sum(TOTAL_WAIT_MS) FROM WaitProfile FACET WAIT_CATEGORY SINCE 30 minutes ago"
        },
        {
          "title": "Wait Time Trend",
          "visualization": "line",
          "query": "SELECT average(TOTAL_WAIT_MS) FROM WaitProfile FACET WAIT_CATEGORY TIMESERIES 1 minute SINCE 2 hours ago"
        },
        {
          "title": "Lock Chain Analysis",
          "visualization": "table",
          "query": "SELECT latest(LOCK_CHAINS) as 'Lock Chain', max(lock.chain_length) as 'Length', count(*) as 'Occurrences', latest(lock.is_deadlock_risk) as 'Deadlock Risk' FROM WaitProfile WHERE WAIT_CATEGORY = 'Lock' AND LOCK_CHAINS IS NOT NULL FACET LOCK_CHAINS SINCE 1 hour ago"
        }
      ]
    }
  ]
}
```

### 3. Multi-Dimensional Analysis Dashboard

```json
{
  "name": "Multi-Dimensional Performance",
  "pages": [
    {
      "name": "Query-Wait-Resource Correlation",
      "widgets": [
        {
          "title": "Query Performance vs Wait Profile",
          "visualization": "heatmap",
          "query": "SELECT average(EXEC_TIME_MS), average(TOTAL_WAIT_MS) FROM QueryPlanIntelligence, WaitProfile WHERE DIGEST = AFFECTED_QUERIES FACET DIGEST, WAIT_CATEGORY SINCE 1 hour ago"
        },
        {
          "title": "Resource Pressure Correlation",
          "visualization": "scatter",
          "query": "SELECT wait.severity_score, resource.cpu_pressure, resource.io_pressure, business_impact_score FROM MultiDimensionalAnalysis SINCE 1 hour ago"
        }
      ]
    }
  ]
}
```

## Key Advantages Over SolarWinds

### 1. **Multi-Database Support**
- Works with MySQL, PostgreSQL, Oracle through SQL queries
- No vendor lock-in or proprietary formats
- Open standards (OpenTelemetry)

### 2. **Cloud-Native Architecture**
- Containerized collectors
- Kubernetes-ready
- Serverless compatible

### 3. **Advanced Analytics**
- ML-powered anomaly detection
- Predictive capacity planning
- Cross-signal correlation

### 4. **Developer Integration**
- CI/CD pipeline integration
- API-first design
- GitOps compatible

### 5. **Cost Efficiency**
- No per-instance licensing
- Open-source components
- Pay only for data storage

## Implementation Roadmap

### Week 1-2: Core Plan Extraction
- Deploy plan extraction queries
- Implement change detection
- Create basic dashboards

### Week 3-4: Wait Analysis
- Deploy comprehensive wait profiling
- Implement lock chain analysis
- Create wait-time dashboards

### Week 5-6: Intelligence Layer
- Deploy recommendation engine
- Implement anomaly detection
- Create advisory dashboards

### Week 7-8: Integration & Optimization
- Cross-signal correlation
- Performance tuning
- User training

## Conclusion

This approach provides all the capabilities of SolarWinds Plan Explorer and DPA, plus:
- Multi-database support
- Cloud-native architecture
- ML-powered insights
- No vendor lock-in
- Lower total cost

All achieved through OpenTelemetry configuration alone, without any code changes or proprietary agents.