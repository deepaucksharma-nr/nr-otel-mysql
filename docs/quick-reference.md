# Quick Reference: Database Intelligence Monorepo

## Essential Commands

### Deployment
```bash
# Quick start (core modules)
cd database-intelligence-monorepo
make run-core-metrics
make run-sql-intelligence
make run-wait-profiler

# Quick start (all modules)
make run-all

# Check status
make health

# View logs for specific module
make logs-sql-intelligence
make logs-wait-profiler
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

### Core Metrics Module (Port 8081)
| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `mysql_connections_current` | Active connections | > 80% max_connections |
| `mysql_threads_running` | Running threads | > 50 |
| `mysql_slow_queries_total` | Slow query count | Increasing trend |
| `mysql_buffer_pool_pages_dirty` | Dirty pages | > 50% |

### SQL Intelligence Module (Port 8082)
| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `mysql_statement_avg_timer_wait` | Average query time | > 1s |
| `mysql_statement_executions` | Query execution count | Monitor top queries |
| `mysql_index_io_wait_count` | Index wait events | > 1000/sec |
| `mysql_table_io_wait_count` | Table I/O wait events | > 5000/sec |

### Anomaly Detector Module (Port 8084)
| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `anomaly_score_cpu` | CPU anomaly score | > 70 |
| `anomaly_score_memory` | Memory anomaly score | > 70 |
| `anomaly_score_connections` | Connection anomaly score | > 80 |
| `anomaly_detected` | Boolean anomaly flag | = 1 |

### Performance Advisor Module (Port 8087)
| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `db.performance.recommendation.missing_index` | Missing index recommendations | Any occurrence |
| `db.performance.recommendation.slow_query` | Slow query recommendations | Any occurrence |
| `db.performance.recommendation.connection_pool` | Pool sizing recommendations | Any occurrence |
| `db.performance.recommendation.cache_efficiency` | Cache recommendations | Any occurrence |

## New Relic Dashboard Queries

### Core Performance Overview
```nrql
SELECT average(mysql_connections_current), 
       average(mysql_threads_running),
       rate(mysql_slow_queries_total, 1 minute)
FROM Metric 
WHERE service.name = 'core-metrics'
TIMESERIES 5 minutes 
SINCE 1 hour ago
```

### SQL Intelligence Analysis
```nrql
SELECT average(mysql_statement_avg_timer_wait) as 'Avg Query Time',
       sum(mysql_statement_executions) as 'Total Executions'
FROM Metric 
WHERE service.name = 'sql-intelligence'
FACET statement_digest
SINCE 1 hour ago
```

### Anomaly Detection Status
```nrql
SELECT latest(anomaly_detected) as 'Anomaly Status',
       max(anomaly_score_cpu) as 'CPU Score',
       max(anomaly_score_memory) as 'Memory Score'
FROM Metric 
WHERE service.name = 'anomaly-detector'
TIMESERIES 1 minute 
SINCE 30 minutes ago
```

### Performance Recommendations
```nrql
SELECT count(*) as 'Recommendation Count'
FROM Metric 
WHERE metricName LIKE 'db.performance.recommendation.%'
FACET metricName, attributes.severity
SINCE 1 day ago
```

### Business Impact Assessment
```nrql
SELECT latest(business_impact_score),
       latest(revenue_impact_hourly)
FROM Metric 
WHERE service.name = 'business-impact'
AND business_criticality = 'CRITICAL'
TIMESERIES 5 minutes 
SINCE 2 hours ago
```

## Module Operations

### Individual Module Commands
```bash
# Start specific modules
make run-core-metrics        # Port 8081
make run-sql-intelligence    # Port 8082
make run-wait-profiler      # Port 8083
make run-anomaly-detector   # Port 8084
make run-business-impact    # Port 8085
make run-replication-monitor # Port 8086
make run-performance-advisor # Port 8087
make run-resource-monitor   # Port 8088

# Stop modules
make stop-core-metrics
make stop-sql-intelligence

# View logs
make logs-anomaly-detector
make logs-performance-advisor

# Clean up
make clean-core-metrics
make docker-clean
```

### Module Groups
```bash
# Run related modules together
make run-core         # core-metrics + resource-monitor
make run-intelligence # sql-intelligence + wait-profiler + anomaly-detector
make run-business     # business-impact + performance-advisor

# Integration testing
make integration      # Full integration test suite
make perf-test       # Performance testing
```

### Health Checks
```bash
# Check all module health
make health

# Individual health checks
curl http://localhost:8081/metrics  # core-metrics
curl http://localhost:8084/metrics  # anomaly-detector
curl http://localhost:13133/       # Health endpoint (most modules)
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
# Check module metrics endpoints
curl -s http://localhost:8081/metrics | grep mysql_  # core-metrics
curl -s http://localhost:8082/metrics | grep mysql_  # sql-intelligence
curl -s http://localhost:8084/metrics | grep anomaly_ # anomaly-detector

# Test MySQL connection from any module
docker exec -it core-metrics-otel-collector \
  mysql -h$MYSQL_ENDPOINT -u$MYSQL_USER -p$MYSQL_PASSWORD -e "SELECT 1"

# Verify Performance Schema
mysql -h$MYSQL_ENDPOINT -u$MYSQL_USER -p$MYSQL_PASSWORD \
  -e "SELECT * FROM performance_schema.setup_consumers WHERE ENABLED='NO'"

# Debug mode for specific module
cd modules/sql-intelligence
docker-compose run --rm sql-intelligence-otel-collector \
  --config /etc/otel/collector.yaml \
  --set service.telemetry.logs.level=debug

# Check module dependencies
make check-dependencies-anomaly-detector
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

## Alert Thresholds by Module

| Module | Metric | Warning | Critical | Action |
|--------|--------|---------|----------|---------|
| Core Metrics | `mysql_connections_current` | > 80% max | > 95% max | Scale connections |
| Core Metrics | `mysql_threads_running` | > 50 | > 100 | Check query load |
| SQL Intelligence | `mysql_statement_avg_timer_wait` | > 1s | > 5s | Optimize queries |
| Anomaly Detector | `anomaly_score_*` | > 50 | > 70 | Investigate anomaly |
| Performance Advisor | Recommendation count | > 5 | > 20 | Review recommendations |
| Business Impact | `revenue_impact_hourly` | > $1000 | > $5000 | Escalate immediately |
| Replication Monitor | `mysql_replica_lag` | > 60s | > 300s | Check replication |
| Resource Monitor | CPU usage | > 80% | > 95% | Scale resources |

## Quick Links

- [Project Overview](../CLAUDE.md) - Complete project documentation
- [Deployment Guide](./plan-intelligence-deployment-guide.md)
- [Use Cases & Examples](./plan-intelligence-use-cases.md)
- [Environment Variables](./ENVIRONMENT_VARIABLES.md)
- [Metric Naming Standards](./METRIC_NAMING_STANDARD.md)
- [New Relic Dashboards](../shared/newrelic/dashboards/)
- [Module Configurations](../modules/)
- [Root Makefile](../Makefile) - All available commands