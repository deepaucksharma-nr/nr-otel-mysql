# MySQL Plan Intelligence & DPA-Style Wait Analysis Deployment Guide

This guide covers deployment and usage of the SolarWinds Plan Explorer and DPA equivalent capabilities built using OpenTelemetry configuration.

## Overview

We've implemented two powerful analysis capabilities that rival SolarWinds' offerings:

1. **Plan Intelligence** (Plan Explorer equivalent) - Query plan analysis, index optimization, and statistics monitoring
2. **Wait Profiler DPA-Style** - Multi-dimensional wait analysis, lock chain detection, and performance correlation

## Architecture

```
┌─────────────────────────┐     ┌──────────────────────┐
│   Plan Intelligence     │     │  Wait Profiler DPA   │
│  (Query Plan Analysis)  │     │  (Wait-Time Method)  │
├─────────────────────────┤     ├──────────────────────┤
│ • Query plan extraction │     │ • Real-time waits    │
│ • Index usage analysis  │     │ • Lock chain detect  │
│ • Statistics monitoring │     │ • Query correlation  │
│ • Recommendations      │     │ • Multi-dimensional  │
└──────────┬──────────────┘     └──────────┬───────────┘
           │                                │
           └────────────┬───────────────────┘
                        ▼
              ┌─────────────────┐
              │   New Relic     │
              │   OTLP Endpoint │
              └────────┬────────┘
                       ▼
              ┌─────────────────┐
              │ Plan Explorer   │
              │   Dashboard     │
              │   (6 pages)     │
              └─────────────────┘
```

## Prerequisites

1. MySQL 8.0+ with Performance Schema enabled
2. OpenTelemetry Collector 0.88.0+
3. New Relic account with OTLP endpoint access
4. Docker and Docker Compose (for containerized deployment)

## Environment Variables

Create a `.env` file with the following configuration:

```bash
# MySQL Connection
MYSQL_ENDPOINT=your-mysql-host:3306
MYSQL_USER=monitoring_user
MYSQL_PASSWORD=secure_password

# New Relic Configuration
NEW_RELIC_LICENSE_KEY=your-license-key
NEW_RELIC_OTLP_ENDPOINT=https://otlp.nr-data.net
NEW_RELIC_ACCOUNT_ID=your-account-id
CLUSTER_NAME=production-mysql-01

# Optional Performance Tuning
OTEL_MEMORY_LIMIT=2Gi
OTEL_CPU_LIMIT=2
```

## MySQL User Setup

Create a monitoring user with appropriate permissions:

```sql
-- Create monitoring user
CREATE USER 'monitoring_user'@'%' IDENTIFIED BY 'secure_password';

-- Grant necessary permissions
GRANT SELECT ON performance_schema.* TO 'monitoring_user'@'%';
GRANT SELECT ON information_schema.* TO 'monitoring_user'@'%';
GRANT PROCESS ON *.* TO 'monitoring_user'@'%';
GRANT REPLICATION CLIENT ON *.* TO 'monitoring_user'@'%';

-- Ensure Performance Schema is enabled
UPDATE performance_schema.setup_instruments 
SET ENABLED = 'YES', TIMED = 'YES' 
WHERE NAME LIKE 'wait/%' OR NAME LIKE 'statement/%';

UPDATE performance_schema.setup_consumers 
SET ENABLED = 'YES' 
WHERE NAME IN (
  'events_waits_current',
  'events_waits_history',
  'events_statements_current',
  'events_statements_history_long',
  'events_waits_summary_by_instance'
);

FLUSH PRIVILEGES;
```

## Deployment Options

### Option 1: Docker Compose (Recommended)

1. **Deploy Plan Intelligence Module:**

```bash
cd modules/sql-intelligence
docker-compose up -d --build
```

2. **Deploy Wait Profiler Module:**

```bash
cd modules/wait-profiler
docker-compose up -d --build
```

3. **Verify deployment:**

```bash
# Check health endpoints
curl http://localhost:13133/health  # Plan Intelligence
curl http://localhost:13134/health  # Wait Profiler

# View logs
docker-compose logs -f
```

### Option 2: Kubernetes Deployment

Create a ConfigMap for each collector configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: plan-intelligence-config
  namespace: monitoring
data:
  collector.yaml: |
    # Contents of collector-plan-intelligence.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: wait-profiler-config
  namespace: monitoring
data:
  collector.yaml: |
    # Contents of collector-dpa-style.yaml
```

Deploy using StatefulSets:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-plan-intelligence
  namespace: monitoring
spec:
  serviceName: plan-intelligence
  replicas: 1
  template:
    spec:
      containers:
      - name: otel-collector
        image: otel/opentelemetry-collector-contrib:0.88.0
        args: ["--config=/etc/otel/collector.yaml"]
        volumeMounts:
        - name: config
          mountPath: /etc/otel
        env:
        - name: MYSQL_ENDPOINT
          valueFrom:
            secretKeyRef:
              name: mysql-credentials
              key: endpoint
        # ... other env vars
      volumes:
      - name: config
        configMap:
          name: plan-intelligence-config
```

### Option 3: Standalone Binary

```bash
# Download OpenTelemetry Collector
wget https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.88.0/otelcol-contrib_0.88.0_linux_amd64.tar.gz
tar -xvf otelcol-contrib_0.88.0_linux_amd64.tar.gz

# Run Plan Intelligence
MYSQL_ENDPOINT=localhost:3306 \
MYSQL_USER=monitoring_user \
MYSQL_PASSWORD=secure_password \
NEW_RELIC_LICENSE_KEY=your-key \
NEW_RELIC_OTLP_ENDPOINT=https://otlp.nr-data.net \
./otelcol-contrib --config=collector-plan-intelligence.yaml

# Run Wait Profiler (in another terminal)
MYSQL_ENDPOINT=localhost:3306 \
MYSQL_USER=monitoring_user \
MYSQL_PASSWORD=secure_password \
NEW_RELIC_LICENSE_KEY=your-key \
NEW_RELIC_OTLP_ENDPOINT=https://otlp.nr-data.net \
./otelcol-contrib --config=collector-dpa-style.yaml
```

## Dashboard Setup

1. **Import Dashboard to New Relic:**

```bash
# Use the provided setup script
cd shared/newrelic/scripts
./setup-newrelic.sh --import-dashboard ../dashboards/plan-explorer-dashboard.json
```

Or manually import through New Relic UI:
- Navigate to Dashboards > Import dashboard
- Paste contents of `plan-explorer-dashboard.json`
- Replace `${NEW_RELIC_ACCOUNT_ID}` with your account ID

2. **Dashboard Pages Overview:**

- **Page 1: Query Plan Analysis** - Plan Explorer equivalent with query performance overview, index effectiveness heatmap, cost analysis
- **Page 2: DPA Wait-Time Analysis** - Wait profile overview, anomaly scoring, trend analysis, lock chains
- **Page 3: Multi-Dimensional Analysis** - Correlate queries, waits, resources, and business impact
- **Page 4: Intelligent Recommendations** - Actionable insights from plan and wait analysis
- **Page 5: Index & Statistics Analysis** - Deep dive into index usage and table statistics
- **Page 6: Real-Time Monitoring** - Live view of current performance issues

## Usage Guide

### 1. Identifying Slow Queries

Navigate to **Query Plan Analysis** page:
- Look for queries with low Index Efficiency Score (<50)
- Check queries marked as "critical" or "slow" in Performance Tier
- Review Missing Index Recommendations table

### 2. Analyzing Wait Events

Navigate to **DPA Wait-Time Analysis** page:
- Monitor Wait Profile Overview pie chart for dominant wait categories
- Check Wait Anomaly Score gauge (>70 indicates critical issues)
- Review Lock Chain Analysis for blocking scenarios

### 3. Correlating Issues

Navigate to **Multi-Dimensional Analysis** page:
- Use Query Performance vs Wait Profile Matrix to identify patterns
- Check Wait Impact by Business Function to prioritize fixes
- Review Resource Pressure Correlation scatter plot

### 4. Taking Action

Navigate to **Intelligent Recommendations** page:
- Follow Top Optimization Opportunities in priority order
- Review Estimated Performance Improvement metrics
- Check specific recommendations for indexes and statistics

## Performance Tuning

### 1. Collector Optimization

```yaml
# In collector configuration
processors:
  memory_limiter:
    check_interval: 5s
    limit_percentage: 80  # Adjust based on available memory
    
  batch:
    timeout: 5s
    send_batch_size: 1000  # Increase for higher throughput
```

### 2. Query Collection Intervals

Adjust based on your environment:
- Plan analysis: 30s (default) - Increase to 60s for large environments
- Wait analysis: 5s (default) - Critical for real-time detection
- Table statistics: 300s (default) - Can increase to 600s
- Index usage: 60s (default) - Stable at this interval

### 3. MySQL Performance Schema Tuning

```sql
-- Increase history lengths for better correlation
UPDATE performance_schema.setup_instruments 
SET ENABLED = 'YES', TIMED = 'YES';

-- Increase statement history
SET GLOBAL performance_schema_events_statements_history_long_size = 10000;

-- Enable more wait event collection
UPDATE performance_schema.setup_consumers 
SET ENABLED = 'YES' WHERE NAME LIKE 'events_waits%';
```

## Troubleshooting

### Common Issues

1. **No data in dashboards:**
   - Verify collectors are running: `docker ps` or `kubectl get pods`
   - Check collector logs for errors
   - Ensure MySQL user has proper permissions
   - Verify Performance Schema is enabled

2. **High memory usage:**
   - Adjust memory_limiter settings
   - Increase collection intervals
   - Reduce batch sizes

3. **Missing recommendations:**
   - Ensure queries have executed >10 times
   - Check that Performance Schema history is retained
   - Verify transform processors are not in error mode

### Debug Mode

Enable debug logging:

```yaml
service:
  telemetry:
    logs:
      level: debug
      
exporters:
  debug:
    verbosity: detailed
```

### Health Checks

Monitor collector health:

```bash
# Plan Intelligence
curl http://localhost:13133/metrics | grep -E "(up|dropped|failed)"

# Wait Profiler  
curl http://localhost:13134/metrics | grep -E "(up|dropped|failed)"
```

## Advanced Features

### 1. Multi-Tenant Support

Configure tenant routing:

```yaml
processors:
  routing:
    from_attribute: "mysql.schema"
    table:
      - value: "production_*"
        exporters: [otlphttp/newrelic_prod]
      - value: "staging_*"
        exporters: [otlphttp/newrelic_staging]
```

### 2. Alert Configuration

Create alerts based on intelligence metrics:

```nrql
# Critical query performance
SELECT count(*) FROM MySQLPlanIntelligence 
WHERE query.performance_tier = 'critical' 
AND query.cost_score > 1000

# Lock chain detection
SELECT max(MAX_LOCK_WAIT_MS) FROM MySQLLockChainAnalysis 
WHERE lock.severity IN ('critical', 'deadlock_risk')

# Wait anomalies
SELECT max(wait.anomaly_score) FROM MySQLWaitAnalysis 
WHERE wait.pattern != 'normal'
```

### 3. Custom Recommendations

Extend recommendation logic in transform processors:

```yaml
transform/custom_recommendations:
  metric_statements:
    - context: datapoint
      statements:
        - set(attributes["custom.recommendation"],
              Case(
                # Your custom logic here
                attributes["SPECIFIC_CONDITION"] == true,
                "Your custom recommendation",
                ""
              ))
```

## Best Practices

1. **Start with baselines** - Run for 24-48 hours to establish normal patterns
2. **Focus on business impact** - Prioritize queries affecting critical business functions
3. **Iterative optimization** - Fix one issue at a time and measure impact
4. **Regular reviews** - Schedule weekly reviews of recommendations
5. **Automate responses** - Use alerts to trigger automated remediation

## Support & Resources

- **Documentation**: See `/docs` directory for detailed configuration guides
- **Dashboards**: Additional dashboards in `/shared/newrelic/dashboards`
- **Scripts**: Automation scripts in `/shared/newrelic/scripts`
- **Examples**: Sample queries and use cases in each module's README

## Next Steps

1. Deploy collectors to your environment
2. Import and customize dashboards
3. Set up alerts for critical metrics
4. Review recommendations weekly
5. Track improvement metrics over time

Remember: These tools provide insights equivalent to SolarWinds Plan Explorer and DPA, but the real value comes from acting on the recommendations and continuously optimizing your MySQL performance.