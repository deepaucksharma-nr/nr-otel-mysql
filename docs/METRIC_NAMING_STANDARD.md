# MySQL Intelligence Metric Naming Standard

## Overview

This document defines the metric naming conventions for the MySQL Database Intelligence monitoring system. Consistent naming ensures metrics can be correlated across modules and dashboards.

## Base Namespaces

### Core MySQL Metrics
- `mysql.*` - Core MySQL server metrics from mysql receiver
- `mysql.query.*` - Query performance and analysis metrics
- `mysql.wait.*` - Wait event and contention metrics
- `mysql.table.*` - Table-level I/O and access metrics
- `mysql.index.*` - Index usage and efficiency metrics
- `mysql.replication.*` - Replication health and lag metrics
- `mysql.innodb.*` - InnoDB-specific metrics
- `mysql.buffer.*` - Buffer pool metrics
- `mysql.lock.*` - Lock wait and contention metrics

### Derived Intelligence Metrics
- `mysql.anomaly.*` - Anomaly detection results
- `mysql.business.*` - Business impact scores
- `mysql.performance.*` - Performance recommendations
- `mysql.resource.*` - Resource utilization metrics

## Metric Type Suffixes

### Counters (Cumulative)
- `*_total` - Total count since start (e.g., mysql.query.exec_total)
- `*_count` - Event occurrences (e.g., mysql.wait.count)

### Gauges (Current Value)
- `*_current` - Current instantaneous value (e.g., mysql.connections.current)
- `*_active` - Currently active items (e.g., mysql.threads.active)
- `*_percent` - Percentage values 0-100 (e.g., mysql.buffer.hit_percent)

### Measurements
- `*_bytes` - Size in bytes (e.g., mysql.innodb.data_bytes)
- `*_ms` - Time in milliseconds (e.g., mysql.query.latency_ms)
- `*_us` - Time in microseconds (e.g., mysql.wait.time_us)
- `*_seconds` - Time in seconds (e.g., mysql.uptime_seconds)

### Rates
- `*_rate` - Rate per second (e.g., mysql.query.exec_rate)
- `*_per_sec` - Operations per second (e.g., mysql.operations.select_per_sec)

## Module Attributes

All metrics MUST include these attributes:
- `module` - Source module name (e.g., "core-metrics", "sql-intelligence")
- `mysql.endpoint` - MySQL instance identifier
- `service.name` - Service name (default: "mysql-intelligence")

## Specific Metric Definitions

### Core Metrics (mysql.*)
```
mysql.connections.current - Current active connections
mysql.connections.max - Maximum connections allowed
mysql.connections.aborted - Aborted connection attempts
mysql.threads.running - Currently executing threads
mysql.threads.connected - Connected threads
mysql.operations.select_total - Total SELECT operations
mysql.operations.insert_total - Total INSERT operations
mysql.operations.update_total - Total UPDATE operations
mysql.operations.delete_total - Total DELETE operations
```

### Query Metrics (mysql.query.*)
```
mysql.query.exec_total - Total query executions
mysql.query.latency_ms - Query execution time in milliseconds
mysql.query.latency_avg_ms - Average query latency
mysql.query.latency_max_ms - Maximum query latency
mysql.query.rows_examined_total - Total rows examined
mysql.query.rows_sent_total - Total rows returned
mysql.query.no_index_used_total - Queries not using indexes
mysql.query.slow_total - Slow query count
```

### Wait Metrics (mysql.wait.*)
```
mysql.wait.count - Wait event occurrences
mysql.wait.time_ms - Wait time in milliseconds
mysql.wait.time_avg_ms - Average wait time
mysql.wait.time_max_ms - Maximum wait time
```

### Table Metrics (mysql.table.*)
```
mysql.table.io.read.latency_ms - Table read latency
mysql.table.io.write.latency_ms - Table write latency
mysql.table.io.read.count - Table read operations
mysql.table.io.write.count - Table write operations
mysql.table.rows.read_total - Rows read from tables
mysql.table.rows.inserted_total - Rows inserted
mysql.table.rows.updated_total - Rows updated
mysql.table.rows.deleted_total - Rows deleted
```

### InnoDB Metrics (mysql.innodb.*)
```
mysql.innodb.buffer_pool.pages.data - Data pages in buffer pool
mysql.innodb.buffer_pool.pages.free - Free pages in buffer pool
mysql.innodb.buffer_pool.hit_rate - Buffer pool hit rate percentage
mysql.innodb.rows.read_total - Total rows read
mysql.innodb.rows.inserted_total - Total rows inserted
mysql.innodb.data_bytes - InnoDB data size in bytes
```

### Anomaly Metrics (mysql.anomaly.*)
```
mysql.anomaly.score - Anomaly score (0-100)
mysql.anomaly.detected - Binary flag for anomaly (0/1)
mysql.anomaly.severity - Severity level (1-5)
```

### Business Metrics (mysql.business.*)
```
mysql.business.impact_score - Business impact score (0-10)
mysql.business.revenue_risk - Revenue risk score
mysql.business.sla_violation - SLA violation flag
```

## Attribute Naming

### Standard Attributes
- `database` - Database name
- `table` - Table name
- `index` - Index name
- `query_digest` - Query digest/fingerprint
- `event_name` - Wait event name
- `wait_category` - Wait event category (io/lock/mutex/network)
- `severity` - Severity level (critical/warning/normal)
- `business_category` - Business category (revenue/customer/analytics)

### Module-Specific Attributes
- `anomaly.type` - Type of anomaly detected
- `anomaly.baseline_source` - Source of baseline (static/dynamic)
- `business.table_type` - Business table classification
- `performance.recommendation` - Performance recommendation type

## Implementation Guidelines

1. **Consistency**: Always use the defined prefixes and suffixes
2. **Units**: Always include units in metric names (ms, bytes, percent)
3. **Clarity**: Metric names should be self-documenting
4. **Attributes**: Use attributes for dimensions, not metric name variations
5. **Case**: Use lowercase with underscores (snake_case)

## Migration from Current State

### Current Inconsistencies to Fix:
- `mysql_query_exec_count` → `mysql.query.exec_total`
- `mysql_wait_count` → `mysql.wait.count`
- `mysql_connections_current` → `mysql.connections.current`
- `total_latency_ms` → `mysql.query.latency_ms`

### Module Mappings:
- core-metrics: Already uses `mysql.*` prefix ✓
- sql-intelligence: Change to `mysql.query.*`, `mysql.table.*`
- wait-profiler: Change to `mysql.wait.*`, `mysql.lock.*`
- anomaly-detector: Add `mysql.anomaly.*` prefix
- business-impact: Add `mysql.business.*` prefix

## Validation

Use the included validation script to check metric naming compliance:
```bash
./scripts/validate-metrics.sh <module-name>
```