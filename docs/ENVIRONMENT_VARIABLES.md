# Environment Variables Configuration Guide

This document lists all configurable environment variables across the Database Intelligence modules.

## Replication Monitor Module

### MySQL Configuration
- `MYSQL_ROOT_PASSWORD` - MySQL root password (default: `rootpassword`)
- `MYSQL_DATABASE` - Default database name (default: `test_db`)
- `MYSQL_REPLICATION_USER` - Replication user name (default: `replication_user`)
- `MYSQL_REPLICATION_PASSWORD` - Replication user password (default: `replication_password`)

### Port Configuration
- `MASTER_PORT` - MySQL master external port (default: `3306`)
- `REPLICA_PORT` - MySQL replica external port (default: `8086`)

### Collector Configuration
- `COLLECTOR_CONFIG` - Collector configuration file name (default: `collector.yaml`)

## Anomaly Detector Module

### Collection Configuration
- `ANOMALY_SCRAPE_INTERVAL` - Prometheus scrape interval (default: `10s`)
- `ANOMALY_BATCH_TIMEOUT` - Batch processor timeout (default: `5s`)
- `ANOMALY_BATCH_SIZE` - Batch processor size (default: `1000`)

### Federation Endpoints
- `CORE_METRICS_ENDPOINT` - Core metrics federation endpoint (default: `core-metrics:8888`)
- `SQL_INTELLIGENCE_ENDPOINT` - SQL intelligence federation endpoint (default: `sql-intelligence:8888`)
- `WAIT_PROFILER_ENDPOINT` - Wait profiler federation endpoint (default: `wait-profiler:8888`)

### Anomaly Detection Thresholds
These thresholds are based on z-scores (standard deviations from mean):
- `CONNECTION_SPIKE_THRESHOLD` - Connection spike detection threshold (default: `3.0`)
- `LATENCY_DEVIATION_THRESHOLD` - Query latency deviation threshold (default: `3.5`)
- `WAIT_EVENT_THRESHOLD` - Wait event anomaly threshold (default: `3.0`)
- `RESOURCE_USAGE_THRESHOLD` - Resource usage anomaly threshold (default: `2.5`)

### Baseline Configuration
#### Connection Baselines
- `CONNECTIONS_BASELINE_MEAN` - Average number of connections (default: `100`)
- `CONNECTIONS_BASELINE_STDDEV` - Standard deviation of connections (default: `20`)

#### Query Duration Baselines
- `QUERY_DURATION_BASELINE_MEAN` - Average query duration in milliseconds (default: `50`)
- `QUERY_DURATION_BASELINE_STDDEV` - Standard deviation of query duration (default: `15`)

#### Wait Time Baselines
- `WAIT_TIME_BASELINE_MEAN` - Average wait time in microseconds (default: `1000`)
- `WAIT_TIME_BASELINE_STDDEV` - Standard deviation of wait time (default: `300`)

#### CPU Usage Baselines
- `CPU_BASELINE_MEAN` - Average CPU usage percentage (default: `50`)
- `CPU_BASELINE_STDDEV` - Standard deviation of CPU usage (default: `10`)

### Service Configuration
- `OTEL_SERVICE_NAME` - OpenTelemetry service name (default: `anomaly-detector`)
- `EXPORT_PORT` - Prometheus exporter port (default: `8889`)

## Business Impact Module

### Impact Scoring Configuration
Impact scores range from 0-10, with 10 being the highest business impact:
- `REVENUE_IMPACT_SCORE` - Score for revenue-impacting queries (default: `10.0`)
- `CUSTOMER_IMPACT_SCORE` - Score for customer-facing operations (default: `8.0`)
- `OPERATIONS_IMPACT_SCORE` - Score for inventory/fulfillment operations (default: `7.0`)
- `ANALYTICS_IMPACT_SCORE` - Score for analytics/reporting queries (default: `5.0`)
- `ADMIN_IMPACT_SCORE` - Score for administrative operations (default: `3.0`)

### Boost Factors
Multipliers applied to scores for critical operations (UPDATE, DELETE, INSERT):
- `REVENUE_BOOST_FACTOR` - Boost factor for revenue category operations (default: `1.5`)
- `CUSTOMER_BOOST_FACTOR` - Boost factor for customer category operations (default: `1.3`)

### Score Limits
- `MAX_IMPACT_SCORE` - Maximum allowed impact score (default: `10.0`)

### SLA Configuration
Duration thresholds in seconds for SLA impact classification:
- `SLA_CRITICAL_THRESHOLD` - Duration above which queries are marked as critical (default: `5.0`)
- `SLA_WARNING_THRESHOLD` - Duration above which queries are marked as warning (default: `2.0`)

### Service Configuration
- `OTEL_SERVICE_NAME` - OpenTelemetry service name (default: `business-impact-analyzer`)

## Best Practices

1. **Environment-Specific Configuration**: Use separate `.env` files for different environments (development, staging, production).

2. **Baseline Tuning**: The baseline values should be calculated from your actual workload patterns. The defaults are examples and should be replaced with values derived from your historical data.

3. **Threshold Adjustment**: Start with the default thresholds and adjust based on false positive/negative rates in your environment.

4. **Security**: Never commit `.env` files with actual passwords to version control. Use secrets management systems in production.

5. **Monitoring**: Monitor the effectiveness of your thresholds and adjust them periodically based on your workload patterns.

## Example .env File

```bash
# Copy the appropriate .env.example file and modify values as needed
cp modules/anomaly-detector/.env.example modules/anomaly-detector/.env

# Edit the file with your specific values
vim modules/anomaly-detector/.env
```