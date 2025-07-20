# Plan Intelligence & DPA Use Cases Guide

Real-world scenarios demonstrating how to use the SolarWinds-equivalent capabilities for MySQL performance optimization.

## Use Case 1: Identifying and Fixing Missing Indexes

### Scenario
Your e-commerce application experiences slow checkout process during peak hours.

### Investigation Steps

1. **Navigate to Query Plan Analysis Dashboard**
   - Sort by "Cost Score" descending
   - Look for queries with "No Index" = YES

2. **Identify the Problem Query**
   ```sql
   -- Example problematic query found
   SELECT o.*, c.name, c.email 
   FROM orders o 
   JOIN customers c ON o.customer_id = c.id 
   WHERE o.created_at > DATE_SUB(NOW(), INTERVAL 7 DAY) 
     AND o.status = 'pending'
   ```

3. **Review Metrics**
   - Index Efficiency Score: 0
   - Rows Examined: 1,245,000
   - Rows Sent: 423
   - Selectivity: 0.03%

4. **Check Recommendations**
   ```
   CREATE INDEX ON orders - Query examines 1245000 rows but returns only 423 (0.03% selectivity)
   ```

5. **Implement Fix**
   ```sql
   CREATE INDEX idx_orders_status_created 
   ON orders(status, created_at) 
   INCLUDE (customer_id);
   ```

6. **Verify Improvement**
   - Monitor Query Plan Analysis after index creation
   - Index Efficiency Score should increase to 80+
   - Execution time should drop significantly

### Expected Outcome
- Query execution time: 5000ms → 50ms (100x improvement)
- CPU usage during checkout: 80% → 15%
- Customer satisfaction: Increased

## Use Case 2: Detecting and Resolving Lock Contention

### Scenario
Users report intermittent timeouts when updating their profiles.

### Investigation Steps

1. **Check Wait-Time Analysis Dashboard**
   - Look at Wait Profile Overview
   - Notice "Lock_Table" represents 45% of waits

2. **Review Lock Chain Analysis**
   ```
   Thread 1052 waiting 5823ms on users.profiles blocked by Thread 892
   Thread 1144 waiting 4521ms on users.profiles blocked by Thread 892
   Thread 1203 waiting 3011ms on users.profiles blocked by Thread 892
   ```

3. **Identify Blocking Query**
   - Thread 892 running for 8 seconds
   - Query: `UPDATE profiles SET last_login = NOW() WHERE user_id IN (...)`
   - Processing 10,000 users in single transaction

4. **Analyze Wait Patterns**
   - Wait anomaly score: 85 (critical)
   - Pattern identified: "lock_contention"
   - Recommendation: "Review transaction isolation levels and query locking behavior"

5. **Implement Solution**
   ```sql
   -- Break into smaller transactions
   DELIMITER $$
   CREATE PROCEDURE update_last_login_batch()
   BEGIN
     DECLARE done INT DEFAULT FALSE;
     DECLARE batch_start INT DEFAULT 0;
     
     WHILE NOT done DO
       START TRANSACTION;
       UPDATE profiles 
       SET last_login = NOW() 
       WHERE user_id BETWEEN batch_start AND batch_start + 100;
       COMMIT;
       
       SET batch_start = batch_start + 100;
       IF batch_start > 10000 THEN
         SET done = TRUE;
       END IF;
     END WHILE;
   END$$
   DELIMITER ;
   ```

### Expected Outcome
- Lock wait time: 5000ms → <100ms
- Concurrent user updates: Success rate 100%
- No more timeout errors

## Use Case 3: Optimizing I/O Bottlenecks

### Scenario
Report generation takes hours during month-end processing.

### Investigation Steps

1. **Analyze Wait Categories**
   - IO_InnoDB: 72% of total wait time
   - Average wait per I/O: 125ms

2. **Correlate with Query Analysis**
   - Find queries with high "Total Rows Examined"
   - Look for full table scans on large tables

3. **Multi-Dimensional Analysis**
   - Check "Resource Pressure Correlation"
   - High correlation between I/O waits and buffer pool misses

4. **Identify Root Cause**
   ```sql
   -- Problematic query
   SELECT 
     DATE(created_at) as order_date,
     SUM(total_amount) as daily_total,
     COUNT(*) as order_count
   FROM orders
   WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31'
   GROUP BY DATE(created_at)
   ```

5. **Review Table Statistics**
   - Table size: 8.5GB
   - Buffer pool size: 4GB
   - Stats age: 456 hours (stale)

6. **Implement Solutions**
   ```sql
   -- Update statistics
   ANALYZE TABLE orders;
   
   -- Create covering index
   CREATE INDEX idx_orders_reporting 
   ON orders(created_at, total_amount);
   
   -- Consider partitioning
   ALTER TABLE orders 
   PARTITION BY RANGE (YEAR(created_at)) (
     PARTITION p2023 VALUES LESS THAN (2024),
     PARTITION p2024 VALUES LESS THAN (2025),
     PARTITION pmax VALUES LESS THAN MAXVALUE
   );
   ```

### Expected Outcome
- Report generation: 3 hours → 15 minutes
- I/O wait percentage: 72% → 15%
- Buffer pool hit ratio: 45% → 95%

## Use Case 4: Identifying Query Plan Regressions

### Scenario
Application performance degrades after weekly maintenance.

### Investigation Steps

1. **Monitor Plan Change Detection**
   - Sudden spike in "Unique Plans" for same query digest
   - Multiple plan changes detected

2. **Compare Query Performance**
   - Before: Avg 50ms, using index
   - After: Avg 2000ms, full table scan

3. **Check Statistics Freshness**
   - Multiple tables showing "Stats Age > 168 hours"
   - Recommendation: "UPDATE STATISTICS for orders - last updated 192 hours ago"

4. **Root Cause Analysis**
   - Auto-analyze disabled during maintenance
   - Data distribution changed significantly
   - Optimizer making poor choices with stale stats

5. **Fix Implementation**
   ```sql
   -- Update all statistics
   SELECT CONCAT('ANALYZE TABLE ', table_name, ';') 
   FROM information_schema.tables 
   WHERE table_schema = 'production';
   
   -- Enable automatic statistics
   SET GLOBAL innodb_stats_auto_recalc = ON;
   SET GLOBAL innodb_stats_persistent = ON;
   ```

### Expected Outcome
- Query plans revert to optimal paths
- Performance restored to baseline
- Automatic prevention of future occurrences

## Use Case 5: Business Impact Prioritization

### Scenario
Limited DBA resources need to focus on highest-impact optimizations.

### Investigation Steps

1. **Review Intelligent Recommendations Dashboard**
   - Sort by "Business Impact" score
   - Filter Priority >= 70

2. **Analyze Top Issues**
   ```
   1. Priority: 100 - checkout_process query
      - Cost Score: 2500
      - Wait Impact: 1800
      - Business Impact: Critical (revenue-impacting)
   
   2. Priority: 85 - inventory_update query  
      - Cost Score: 1200
      - Lock severity: high
      - Business Impact: High (order fulfillment)
   
   3. Priority: 70 - user_preferences query
      - Index missing
      - Business Impact: Medium (user experience)
   ```

3. **Create Action Plan**
   - Week 1: Fix checkout_process (est. 4hr work, $50K/day impact)
   - Week 2: Optimize inventory_update (est. 2hr work, prevent fulfillment delays)
   - Week 3: Add index for user_preferences (est. 1hr work, improve UX)

### Expected Outcome
- ROI-driven optimization
- Measurable business value delivery
- Efficient resource utilization

## Use Case 6: Real-Time Performance Monitoring

### Scenario
Need to monitor Black Friday traffic in real-time.

### Investigation Steps

1. **Set Up Real-Time Monitoring Dashboard**
   - Focus on "Active Wait Events" chart
   - Monitor "Current Lock Status" billboard
   - Watch "Query Execution Heatmap"

2. **Create Alerts**
   ```nrql
   -- Critical query threshold
   SELECT count(*) FROM MySQLPlanIntelligence 
   WHERE query.performance_tier = 'critical' 
   FACET DIGEST
   
   -- Lock escalation warning
   SELECT max(BLOCKED_THREADS) FROM MySQLLockChainAnalysis
   WHERE timestamp > now() - 60
   ```

3. **Establish Response Procedures**
   - Wait anomaly > 70: Check lock chains
   - Critical queries > 5: Review plan changes
   - I/O waits spike: Check buffer pool

4. **Live Optimization**
   - Kill long-running queries blocking others
   - Adjust connection pool sizes
   - Route read traffic to replicas

### Expected Outcome
- Proactive issue resolution
- Zero downtime during peak traffic
- Optimal performance maintained

## Best Practices from Use Cases

1. **Start with Business Impact**
   - Always consider business value when prioritizing
   - Track improvements in business metrics

2. **Use Multi-Dimensional Analysis**
   - Don't analyze queries in isolation
   - Correlate waits, resources, and queries

3. **Monitor Plan Stability**
   - Track plan changes over time
   - Keep statistics fresh

4. **Automate Common Fixes**
   - Script index creation for repeated patterns
   - Automate statistics updates
   - Build runbooks for common scenarios

5. **Measure Success**
   - Before/after metrics for each optimization
   - Track cumulative improvements
   - Report business value delivered

## Quick Reference: Problem → Dashboard → Action

| Problem | Dashboard Page | Key Metrics | Action |
|---------|---------------|-------------|---------|
| Slow queries | Query Plan Analysis | Index Efficiency Score, Cost Score | Create missing indexes |
| Timeouts | DPA Wait-Time Analysis | Lock chains, Wait anomaly | Break large transactions |
| I/O bottleneck | Multi-Dimensional Analysis | IO waits, Buffer pool | Optimize queries, increase memory |
| Plan regression | Query Plan Analysis | Plan changes, Stats age | Update statistics |
| Resource pressure | Real-Time Monitoring | CPU, Memory correlation | Scale resources, optimize queries |
| Business impact | Intelligent Recommendations | Priority scores | Focus on high-impact fixes |