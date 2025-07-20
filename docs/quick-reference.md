# Quick Reference: MySQL Plan Intelligence & DPA

## Essential Commands

### Deployment
```bash
# Quick start (both modules)
cd database-intelligence-monorepo
docker-compose -f modules/sql-intelligence/docker-compose.yaml up -d
docker-compose -f modules/wait-profiler/docker-compose.yaml up -d

# Check status
curl -s http://localhost:13133/health | jq .  # Plan Intelligence
curl -s http://localhost:13134/health | jq .  # Wait Profiler

# View logs
docker-compose -f modules/sql-intelligence/docker-compose.yaml logs -f
docker-compose -f modules/wait-profiler/docker-compose.yaml logs -f
```

### MySQL Setup
```sql
-- Quick monitoring user setup
CREATE USER 'otel_monitor'@'%' IDENTIFIED BY 'secure_password';
GRANT SELECT ON performance_schema.* TO 'otel_monitor'@'%';
GRANT SELECT ON information_schema.* TO 'otel_monitor'@'%';
GRANT PROCESS, REPLICATION CLIENT ON *.* TO 'otel_monitor'@'%';
FLUSH PRIVILEGES;

-- Enable Performance Schema (required)
UPDATE performance_schema.setup_instruments SET ENABLED = 'YES', TIMED = 'YES';
UPDATE performance_schema.setup_consumers SET ENABLED = 'YES';
```

## Key Metrics Reference

### Plan Intelligence Metrics
| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `INDEX_EFFICIENCY_SCORE` | Query index usage (0-100) | < 50 |
| `query.cost_score` | Combined cost metric | > 1000 |
| `SELECTIVITY` | Rows returned / examined | < 0.1 |
| `query.performance_tier` | critical/slow/acceptable/good/optimal | = critical |
| `stats.is_stale` | Table statistics outdated | = true |
| `index.recommendation_type` | CREATE_INDEX/OPTIMIZE_INDEX/NONE | != NONE |

### Wait Analysis Metrics
| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `wait.anomaly_score` | Wait anomaly detection (0-100) | > 70 |
| `MAX_LOCK_WAIT_MS` | Maximum lock wait time | > 5000 |
| `WAIT_PERCENTAGE` | Query time spent waiting | > 50 |
| `lock.severity` | deadlock_risk/critical/high/medium/low | >= high |
| `wait.pattern` | lock_contention/io_bottleneck/mutex_contention | != normal |

## NRQL Queries Cheat Sheet

### Top Slow Queries
```nrql
SELECT latest(DIGEST_TEXT), average(AVG_TIME_MS), sum(EXECUTION_COUNT) 
FROM MySQLPlanIntelligence 
WHERE query.performance_tier IN ('critical', 'slow')
FACET DIGEST 
SINCE 1 hour ago
```

### Missing Indexes
```nrql
SELECT latest(recommendation.text), latest(PRIMARY_TABLE), sum(ROWS_EXAMINED)
FROM MySQLPlanIntelligence 
WHERE index.recommendation_type = 'CREATE_INDEX'
FACET DIGEST 
SINCE 24 hours ago
```

### Current Lock Chains
```nrql
SELECT latest(LOCK_CHAINS), max(MAX_LOCK_WAIT_MS)
FROM MySQLLockChainAnalysis 
WHERE LOCK_WAIT_COUNT > 0 
SINCE 5 minutes ago
```

### Wait Anomalies
```nrql
SELECT max(wait.anomaly_score), latest(wait.pattern)
FROM MySQLWaitAnalysis 
WHERE wait.anomaly_score > 50
FACET WAIT_CATEGORY 
TIMESERIES 1 minute 
SINCE 30 minutes ago
```

### Query-Wait Correlation
```nrql
SELECT latest(DIGEST_TEXT), average(WAIT_PERCENTAGE), latest(DOMINANT_WAIT_EVENT)
FROM MySQLQueryWaitCorrelation 
WHERE WAIT_PERCENTAGE > 30
FACET DIGEST 
SINCE 1 hour ago
```

## Common Patterns & Solutions

### Pattern: High Lock Waits
**Symptoms:**
- `lock.severity` = critical
- `BLOCKED_THREADS` > 5
- Timeouts in application

**Quick Fix:**
```sql
-- Find blocking queries
SELECT * FROM information_schema.INNODB_LOCKS;
SELECT * FROM information_schema.INNODB_LOCK_WAITS;

-- Kill blocking thread
KILL <thread_id>;
```

### Pattern: Missing Indexes
**Symptoms:**
- `NO_INDEX_USED` = YES
- `INDEX_EFFICIENCY_SCORE` < 30
- High `ROWS_EXAMINED`

**Quick Fix:**
```sql
-- Get index recommendation from dashboard
-- Verify with EXPLAIN
EXPLAIN SELECT ...;

-- Create index
CREATE INDEX idx_name ON table(columns);
```

### Pattern: I/O Bottleneck
**Symptoms:**
- `wait.pattern` = io_bottleneck
- IO_InnoDB > 50% of waits
- Slow full table scans

**Quick Fix:**
```sql
-- Increase buffer pool
SET GLOBAL innodb_buffer_pool_size = 8589934592;  -- 8GB

-- Warm up buffer pool
SELECT COUNT(*) FROM large_table;
```

### Pattern: Stale Statistics
**Symptoms:**
- `stats.is_stale` = true
- Plan changes detected
- Performance regression

**Quick Fix:**
```sql
-- Update specific table
ANALYZE TABLE table_name;

-- Update all tables
mysqlcheck -Aa --auto-repair
```

## Environment Variables Quick Reference

```bash
# Required
export MYSQL_ENDPOINT=mysql-host:3306
export MYSQL_USER=otel_monitor
export MYSQL_PASSWORD=secure_password
export NEW_RELIC_LICENSE_KEY=your-key
export NEW_RELIC_OTLP_ENDPOINT=https://otlp.nr-data.net

# Optional tuning
export OTEL_MEMORY_LIMIT=2Gi
export COLLECTION_INTERVAL=30s
export BATCH_SIZE=1000
```

## Troubleshooting Commands

```bash
# Check collector metrics
curl -s http://localhost:8889/metrics | grep mysql_

# Test MySQL connection
mysql -h$MYSQL_ENDPOINT -u$MYSQL_USER -p$MYSQL_PASSWORD -e "SELECT 1"

# Verify Performance Schema
mysql -h$MYSQL_ENDPOINT -u$MYSQL_USER -p$MYSQL_PASSWORD \
  -e "SELECT * FROM performance_schema.setup_consumers WHERE ENABLED='NO'"

# Debug mode
docker-compose -f modules/sql-intelligence/docker-compose.yaml \
  run --rm sql-intelligence \
  --config /etc/otel/collector.yaml \
  --set service.telemetry.logs.level=debug
```

## Performance Tuning Quick Wins

### 1. Collector Optimization
```yaml
processors:
  batch:
    send_batch_size: 2000  # Increase for better throughput
    timeout: 10s           # Increase for larger batches
```

### 2. MySQL Optimization
```sql
-- Increase Performance Schema memory
SET GLOBAL performance_schema_max_digest_length = 4096;
SET GLOBAL performance_schema_events_statements_history_long_size = 10000;
```

### 3. Query Collection
```yaml
collection_interval: 60s  # Increase for large environments
queries:
  - sql: "... LIMIT 200"  # Increase limits for more coverage
```

## Alert Thresholds

| Metric | Warning | Critical | Action |
|--------|---------|----------|---------|
| Query execution time | > 1s | > 5s | Optimize query |
| Lock wait time | > 1s | > 5s | Review transactions |
| Index efficiency | < 70 | < 50 | Add indexes |
| Wait anomaly score | > 40 | > 70 | Investigate waits |
| Blocked threads | > 2 | > 5 | Check lock chains |
| Stats age | > 7 days | > 14 days | Update statistics |

## Quick Links

- [Full Deployment Guide](./plan-intelligence-deployment-guide.md)
- [Use Cases & Examples](./plan-intelligence-use-cases.md)
- [New Relic Dashboards](../shared/newrelic/dashboards/)
- [Module Configs](../modules/)
- [Troubleshooting Scripts](../shared/newrelic/scripts/)