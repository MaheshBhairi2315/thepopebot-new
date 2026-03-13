# Health Check Metrics Reference Guide

A comprehensive guide to monitoring and alerting on system health across infrastructure, applications, and services.

---

## Quick Reference Summary Table

| Category | Metric | Typical Warning | Typical Critical | Alert Priority |
|----------|--------|-----------------|------------------|----------------|
| **HTTP/API** | Response Time | > 500ms | > 2s | P2/P1 |
| **HTTP/API** | Error Rate (5xx) | > 1% | > 5% | P2/P1 |
| **HTTP/API** | P95 Latency | > 1s | > 5s | P2 |
| **HTTP/API** | Uptime | < 99.9% | < 99% | P1 |
| **HTTP/API** | Throughput | Drop > 30% | Drop > 70% | P2 |
| **System** | CPU Utilization | > 70% | > 90% | P3/P2 |
| **System** | Memory Usage | > 80% | > 95% | P2 |
| **System** | Disk Space | > 80% | > 90% | P2/P1 |
| **System** | Load Average | > 2x cores | > 4x cores | P2 |
| **Database** | Connection Pool | > 70% | > 90% | P2 |
| **Database** | Slow Queries | > 10/min | > 50/min | P3 |
| **Database** | Replication Lag | > 1s | > 10s | P2/P1 |
| **Database** | Deadlocks | > 0 | > 5/hr | P2 |
| **Application** | Queue Depth | > 1000 | > 10,000 | P3/P2 |
| **Application** | Dependency Health | Any failure | Multiple failures | P2/P1 |
| **Application** | Error Log Rate | > 10/min | > 100/min | P3/P2 |
| **Synthetic** | SSL Expiry | < 30 days | < 7 days | P3/P2 |
| **Synthetic** | DNS Resolution | > 100ms | > 1s | P3 |
| **Synthetic** | Regional Reachability | 1 region fail | 2+ regions fail | P2/P1 |

*P1 = Page immediately, P2 = Page during business hours, P3 = Ticket/warn only*

---

## 1. HTTP/API Health Checks

### Response Time
- **What it measures**: Time from request to first byte (TTFB) or full response
- **Typical thresholds**: 
  - Warning: > 200-500ms
  - Critical: > 1-2s
- **Tools**: Prometheus + Grafana, Datadog, New Relic, Pingdom, UptimeRobot, AWS CloudWatch
- **Alerting best practices**: 
  - Use median (p50) for general health, p95/p99 for user experience
  - Alert on sustained elevation (3+ minutes) to avoid noise from spikes
  - Page for p99 > SLA threshold; warn for p50 degradation

### Status Codes / Error Rates
- **What it measures**: Percentage of 5xx errors vs total requests
- **Typical thresholds**:
  - Warning: > 0.1-1% error rate
  - Critical: > 5% error rate
- **Tools**: Prometheus, Datadog, Splunk, ELK Stack, Grafana Loki
- **Alerting best practices**:
  - Never alert on 4xx errors (client errors)
  - Require minimum request volume (e.g., > 100 req/min) to avoid false positives
  - Separate alerts for different endpoint criticality

### Latency Percentiles (p50, p95, p99)
- **What it measures**: Response time distribution at different percentiles
- **Typical thresholds**:
  - p50: < 100ms
  - p95: < 500ms  
  - p99: < 1-2s
- **Tools**: Prometheus histograms, Datadog distributions, HDR Histogram
- **Alerting best practices**:
  - p50 for overall health monitoring (warn only)
  - p95/p99 for SLO compliance and user experience (page if breached)
  - Use sliding windows (e.g., 5-minute avg) to smooth outliers

### Uptime / Availability
- **What it measures**: Percentage of time service is reachable and responding correctly
- **Typical thresholds**:
  - Warning: < 99.9% ("three nines")
  - Critical: < 99% ("two nines")
  - SLO targets: 99.9% (8.76h downtime/year), 99.99% (52.6m/year)
- **Tools**: Pingdom, UptimeRobot, StatusCake, Datadog SLO Monitoring, Grafana
- **Alerting best practices**:
  - Define SLOs with error budgets
  - Burn rate alerts for fast SLO consumption
  - Page immediately for total outage; warn for degraded availability

### Throughput (Requests Per Second)
- **What it measures**: Volume of requests handled per second
- **Typical thresholds**:
  - Warning: Drop > 30% from baseline
  - Critical: Drop > 70% or unexpected spike > 300%
- **Tools**: Prometheus rate() functions, Datadog, AWS CloudWatch, Google Cloud Monitoring
- **Alerting best practices**:
  - Alert on drops (upstream issues) more than spikes (unless DDoS)
  - Compare against same time previous week (seasonality)
  - Correlate with deployment events

---

## 2. System / Infrastructure Metrics

### CPU Utilization
- **What it measures**: Percentage of CPU capacity being used
- **Typical thresholds**:
  - Warning: > 70% sustained
  - Critical: > 90% sustained (> 5 min)
- **Tools**: Prometheus node_exporter, Datadog Agent, CloudWatch, New Relic Infrastructure, Telegraf
- **Alerting best practices**:
  - Alert on sustained high usage, not brief spikes
  - Consider CPU steal time in virtualized environments
  - Distinguish between user, system, and iowait
  - P3 ticket for warnings; P2 page for critical during business hours

### Memory Usage
- **What it measures**: Percentage of RAM consumed (used vs available)
- **Typical thresholds**:
  - Warning: > 80%
  - Critical: > 90-95%
- **Tools**: Prometheus, Datadog, Nagios, Zabbix, Grafana
- **Alerting best practices**:
  - Include buffer/cache in calculations (available vs free)
  - Watch for OOM killer events
  - Memory leaks show as gradual growth over days/weeks
  - P2 page when approaching critical; trend analysis for warnings

### Disk I/O
- **What it measures**: Read/write operations per second, throughput (MB/s), I/O wait
- **Typical thresholds**:
  - Warning: > 80% I/O utilization, > 20ms avg wait
  - Critical: > 95% utilization, > 100ms wait
- **Tools**: iostat, Prometheus node_exporter, Datadog, sysstat
- **Alerting best practices**:
  - Correlate with application latency
  - SSDs handle higher IOPS than thresholds suggest
  - Alert on I/O wait % as indicator of CPU starvation
  - P3 for warnings; P2 if affecting application performance

### Disk Space
- **What it measures**: Percentage of disk capacity used
- **Typical thresholds**:
  - Warning: > 80% full
  - Critical: > 90-95% full
- **Tools**: Prometheus, Nagios, Datadog, CloudWatch, custom scripts
- **Alerting best practices**:
  - Include trend prediction (hours until full)
  - Different thresholds for different volume types (logs vs data)
  - Alert on inode exhaustion separately (common with many small files)
  - P3 ticket at 80%; P2 page at 90% with runbook for cleanup

### Network Latency
- **What it measures**: Round-trip time between nodes, packet loss
- **Typical thresholds**:
  - Warning: > 10ms (internal), > 100ms (external)
  - Critical: > 100ms (internal), > 500ms (external)
  - Packet loss: > 0.1%
- **Tools**: SmokePing, Ping/prometheus_blackbox, Datadog Network Monitor, ThousandEyes
- **Alerting best practices**:
  - Measure between critical service tiers (app→db, app→cache)
  - Baseline varies by geography
  - Packet loss often more indicative than latency
  - P2 for internal network issues; P3 for external/transit

### Load Average
- **What it measures**: Average number of processes waiting for CPU (1, 5, 15 min)
- **Typical thresholds**:
  - Warning: > 2x number of CPU cores
  - Critical: > 4x number of CPU cores
- **Tools**: uptime, top, Prometheus node_exporter, htop
- **Alerting best practices**:
  - Compare 1-min vs 15-min to see trends
  - Different meaning on containers vs bare metal
  - High load + low CPU = I/O or lock contention
  - P3 for warnings; investigate before paging

---

## 3. Database Health

### Connection Pool Status
- **What it measures**: Active connections vs pool capacity, wait queue depth
- **Typical thresholds**:
  - Warning: > 70% of max connections
  - Critical: > 90% or connection wait time > 1s
- **Tools**: Prometheus postgres_exporter/mysql_exporter, pg_stat_activity, Datadog Database Monitoring
- **Alerting best practices**:
  - Monitor connection leaks (growth over time)
  - Include idle connections in assessment
  - Set max_connections appropriately with headroom
  - P2 page when approaching limits; P1 if connections rejected

### Query Performance (Slow Queries)
- **What it measures**: Queries exceeding time threshold, avg execution time
- **Typical thresholds**:
  - Warning: > 10 slow queries/min (>{query_time_threshold}s)
  - Critical: > 50 slow queries/min
- **Tools**: pgBadger, Percona Toolkit, MySQL slow query log, pganalyze, Datadog
- **Alerting best practices**:
  - Define "slow" per query type (OLTP vs OLAP)
  - Track 95th percentile query time, not average
  - Correlate with deployment/schema changes
  - P3 ticket for warnings; P2 if impacting user experience

### Replication Lag
- **What it measures**: Delay between primary and replica (seconds or bytes behind)
- **Typical thresholds**:
  - Warning: > 1-5 seconds lag
  - Critical: > 10-30 seconds lag
- **Tools**: pg_stat_replication, SHOW SLAVE STATUS, Datadog, PMM, custom scripts
- **Alerting best practices**:
  - Critical for read-after-write consistency
  - Alert on lag growth rate, not just absolute value
  - Consider failover implications
  - P2 page for critical lag; P1 if replicas fall significantly behind

### Deadlocks
- **What it measures**: Number of deadlocks per time period
- **Typical thresholds**:
  - Warning: > 0 (any deadlock)
  - Critical: > 5-10 per hour
- **Tools**: PostgreSQL pg_stat_database, MySQL SHOW ENGINE INNODB STATUS, Datadog
- **Alerting best practices**:
  - Any deadlock indicates application bug
  - Log full deadlock details for analysis
  - Trend over time to catch increasing frequency
  - P3 ticket for isolated deadlocks; P2 if frequent

### Cache Hit Ratio
- **What it measures**: Percentage of reads served from cache vs disk
- **Typical thresholds**:
  - Warning: < 95% (PostgreSQL shared_buffers)
  - Critical: < 90%
- **Tools**: pg_stat_database, MySQL Performance Schema, Redis INFO, Datadog
- **Alerting best practices**:
  - Different databases have different normal ranges
  - Sudden drops indicate workload change or cache pressure
  - Buffer cache vs query cache vs application cache
  - P3 for warnings; usually not page-worthy alone

---

## 4. Application-Level Checks

### Dependency Health (Downstream Services)
- **What it measures**: Availability and latency of services the app depends on
- **Typical thresholds**:
  - Warning: Any dependency unhealthy
  - Critical: Critical dependency down or > 50% degraded
- **Tools**: Consul health checks, Kubernetes readiness probes, Datadog Synthetics, custom health endpoints
- **Alerting best practices**:
  - Distinguish critical vs non-critical dependencies
  - Circuit breaker pattern for graceful degradation
  - Health check endpoints that verify deep dependencies
  - P2 for single dependency; P1 for critical path failures

### Queue Depth
- **What it measures**: Number of messages/jobs waiting in queue
- **Typical thresholds**:
  - Warning: > 1000 messages
  - Critical: > 10,000 messages or growing > 100/min
- **Tools**: RabbitMQ Management, AWS SQS metrics, Sidekiq dashboard, Prometheus, Datadog
- **Alerting best practices**:
  - Alert on growth rate, not just absolute depth
  - Different thresholds per queue priority
  - Consider time-in-queue (age of oldest message)
  - P3 for warnings; P2 if processing falling behind

### Error Log Rate
- **What it measures**: Volume of ERROR/FATAL log lines per minute
- **Typical thresholds**:
  - Warning: > 10 errors/min
  - Critical: > 100 errors/min or any FATAL
- **Tools**: ELK Stack, Splunk, Datadog Log Management, Grafana Loki, Fluentd
- **Alerting best practices**:
  - Group by error type to avoid alert storms
  - Exclude known non-actionable errors
  - Different thresholds for ERROR vs WARN vs FATAL
  - P3 for warnings; P2 for elevated error rates

### Business Logic Sanity Checks
- **What it measures**: Domain-specific validations (e.g., order completion rate, payment success rate)
- **Typical thresholds**: Varies by business (e.g., payment success < 95%)
- **Tools**: Custom metrics in application code, Datadog custom metrics, Prometheus client libraries
- **Alerting best practices**:
  - Align with business SLAs
  - Compare to historical baselines
  - Often best early indicator of problems
  - P2 for anomalies; P1 for critical business impact

### Garbage Collection (GC) Metrics
- **What it measures**: GC frequency, pause duration, heap usage growth
- **Typical thresholds**:
  - Warning: > 1s pause time, > 20% time in GC
  - Critical: > 5s pause, memory leak pattern
- **Tools**: JMX exporters, .NET counters, Go pprof, Datadog APM, New Relic
- **Alerting best practices**:
  - Different GC types (Young vs Full) have different thresholds
  - Memory leak = gradual heap growth with same load
  - Correlate GC pauses with latency spikes
  - P3 for warnings; P2 if causing latency

---

## 5. Synthetic / External Monitoring

### Endpoint Reachability (Multi-Region)
- **What it measures**: HTTP success from multiple geographic locations
- **Typical thresholds**:
  - Warning: 1 region failing
  - Critical: 2+ regions failing or > 50% regions
- **Tools**: Pingdom, Datadog Synthetics, New Relic Synthetics, ThousandEyes, AWS Route 53 Health Checks
- **Alerting best practices**:
  - Test from regions matching user base
  - Include API validation (not just 200 OK)
  - Distinguish regional vs global outages
  - P2 for regional; P1 for multi-region or global

### SSL Certificate Expiry
- **What it measures**: Days until TLS certificate expiration
- **Typical thresholds**:
  - Warning: < 30 days
  - Critical: < 7 days
- **Tools**: SSL Labs API, cert-manager, Prometheus blackbox exporter, Datadog
- **Alerting best practices**:
  - Earlier warnings for manual renewal processes
  - Monitor intermediate certificates too
  - Check certificate chain validity
  - P3 at 30 days; P2 at 7 days; P1 at 1 day

### DNS Resolution Time
- **What it measures**: Time to resolve hostname to IP
- **Typical thresholds**:
  - Warning: > 100ms
  - Critical: > 1s or resolution failure
- **Tools**: dig, Prometheus blackbox, Datadog Synthetics, DNSChecker
- **Alerting best practices**:
  - Test against multiple DNS providers
  - Monitor both authoritative and recursive resolution
  - DNS propagation delays after changes
  - P3 for latency; P2 for resolution failures

### Page Load Performance
- **What it measures**: Full page load time, Core Web Vitals (LCP, FID, CLS)
- **Typical thresholds**:
  - LCP: < 2.5s (good), < 4s (needs improvement)
  - FID: < 100ms
  - CLS: < 0.1
- **Tools**: Google PageSpeed Insights, Lighthouse, WebPageTest, Datadog RUM, New Relic Browser
- **Alerting best practices**:
  - Focus on Core Web Vitals for SEO/UX
  - Real User Monitoring (RUM) over synthetic for accuracy
  - 75th percentile of page loads for SLO
  - P3 for degradation; P2 if significantly impacting users

### Transaction Monitoring (Synthetic)
- **What it measures**: End-to-end user journeys (login → purchase)
- **Typical thresholds**: 
  - Warning: > 10s total time
  - Critical: Any step failure or > 30s
- **Tools**: Selenium-based tools, Datadog Synthetics, New Relic Synthetics, Ghost Inspector
- **Alerting best practices**:
  - Run continuously from multiple locations
  - Test critical revenue paths frequently (1-5 min)
  - Isolate failures by step
  - P1 for critical path failures; P2 for degraded performance

---

## General Alerting Best Practices

### Alert Severity Guidelines

| Severity | Response | Examples |
|----------|----------|----------|
| **P1 - Critical** | Immediate page, 24/7 | Service down, data loss, security breach, 100% error rate |
| **P2 - High** | Page during business hours | Degraded performance, single dependency down, high error rates |
| **P3 - Medium** | Ticket/email, next-day response | Elevated warnings, approaching thresholds, non-urgent issues |
| **P4 - Low** | Log only, weekly review | Informational metrics, capacity planning data |

### Key Principles

1. **Signal-to-Noise Ratio**: Every alert should be actionable. If you ignore it, remove it.

2. **Multi-Condition Alerts**: Require multiple conditions (e.g., high CPU + high latency) to reduce false positives.

3. **Burn Rate Alerts**: For SLO-based monitoring, alert on error budget consumption rate.

4. **Runbook Links**: Every alert should link to a runbook with diagnostic steps and remediation.

5. **Graduated Response**: Start with warnings before paging. Escalate based on duration/severity.

6. **Context-Rich Notifications**: Include relevant graphs, logs, and recent deployments in alert context.

7. **Regular Review**: Review and tune thresholds monthly. Remove alerts that never fire.

---

## Recommended Tooling Stack by Scale

| Scale | Recommended Stack | Cost Profile |
|-------|-------------------|--------------|
| **Startup/Small** | Prometheus + Grafana + UptimeRobot + PagerDuty | Low (open source) |
| **Mid-Size** | Datadog or New Relic (APM + Infra + Logs) | Medium (SaaS) |
| **Enterprise** | Hybrid: Prometheus (metrics) + Splunk/ELK (logs) + ThousandEyes (synthetics) + PagerDuty/Opsgenie | High |

---

*Last updated: March 2026*
