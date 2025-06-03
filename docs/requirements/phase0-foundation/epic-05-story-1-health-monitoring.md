# 05.1 - Health Monitoring and Metrics

## User Story
**As an** operations engineer  
**I want** comprehensive monitoring capabilities  
**So that** I can ensure system health and quickly diagnose issues

## INVEST Criteria
- **Independent**: Can be developed alongside other operational features
- **Negotiable**: Specific metrics and thresholds can be adjusted
- **Valuable**: Essential for production operations
- **Estimable**: 1-2 sprints for comprehensive implementation
- **Small**: Focused on monitoring infrastructure
- **Testable**: Health checks and metrics can be verified

## Acceptance Criteria

### Scenario: Component Health Checks
```gherkin
Given a running system with multiple components
When I query GET /health
Then I should receive a comprehensive health status:
  | Component          | Checks                                         | Critical |
  | api               | Process running, memory < 90%, CPU < 80%      | Yes      |
  | database          | Connection pool, query latency < 100ms         | Yes      |
  | cassandra         | Cluster status, all nodes UN                   | Yes      |
  | redis             | Connection available, memory usage             | No       |
  | celery            | Workers running, queue depth < 1000            | No       |
  | disk              | Free space > 20%, inodes > 10%               | Yes      |
  | certificates      | CA cert valid, expires > 30 days              | Yes      |
And the response format should be:
{
  "status": "healthy|degraded|unhealthy",
  "timestamp": "ISO8601",
  "version": "1.0.0",
  "checks": {
    "component": {
      "status": "up|down",
      "latency_ms": 10,
      "details": {}
    }
  }
}
```

### Scenario: Liveness vs Readiness Probes
```gherkin
Given Kubernetes deployments need different health checks
When container orchestration is used
Then provide separate endpoints:
  | Endpoint          | Purpose                    | Checks                        | Failure Action |
  | /health/live     | Is process alive?          | Process responding            | Restart pod    |
  | /health/ready    | Can serve traffic?         | All dependencies available    | Remove from LB |
  | /health/startup  | Is app initialising?       | Startup tasks complete        | Wait longer    |
And configure appropriate timeouts:
  | Probe    | Initial Delay | Period | Timeout | Failures |
  | Liveness | 60s          | 10s    | 5s      | 3        |
  | Readiness| 10s          | 5s     | 3s      | 2        |
  | Startup  | 0s           | 5s     | 10s     | 12       |
```

### Scenario: Prometheus Metrics Export
```gherkin
Given Prometheus is the standard metrics system
When I query GET /metrics
Then return metrics in Prometheus format:
"""
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="POST",endpoint="/certificates",status="200"} 1027

# HELP http_request_duration_seconds HTTP request latency
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.005",endpoint="/health"} 1000
http_request_duration_seconds_bucket{le="0.01",endpoint="/health"} 1500
http_request_duration_seconds_bucket{le="0.025",endpoint="/health"} 1750

# HELP certificate_operations_total Certificate operations by type
# TYPE certificate_operations_total counter
certificate_operations_total{operation="generate",type="node"} 523
certificate_operations_total{operation="generate",type="client"} 1893
certificate_operations_total{operation="revoke",type="all"} 12
"""
And include standard metrics:
  | Metric Category    | Examples                                    |
  | HTTP metrics      | Request rate, latency, errors by endpoint  |
  | Business metrics  | Certificates generated, revoked, expired    |
  | System metrics    | CPU, memory, disk usage                     |
  | Database metrics  | Query latency, connection pool stats        |
  | Queue metrics     | Task completion time, queue depth           |
```

### Scenario: Custom Business Metrics
```gherkin
Given business operations need tracking
When certificate operations occur
Then record domain-specific metrics:
  | Metric                              | Type      | Labels                          |
  | cert_generation_duration_seconds    | histogram | algorithm, key_size             |
  | cert_expiry_days                   | gauge     | type, ca                        |
  | cert_operations_total              | counter   | operation, status, type         |
  | active_certificates                | gauge     | type, ca                        |
  | auth_attempts_total                | counter   | method, status                  |
  | api_key_usage                      | counter   | key_id, endpoint                |
  | ca_hierarchy_depth                 | gauge     | ca_id                          |
  | scb_generations_total              | counter   | status                          |
  | deployment_duration_seconds        | histogram | target_type, status             |
  | audit_events_total                 | counter   | event_type, severity            |
```

### Scenario: Performance Metrics Collection
```gherkin
Given performance monitoring is critical
When operations are performed
Then collect detailed timing metrics:
  | Operation                 | Metrics Collected                              |
  | API request              | Total time, auth time, processing time         |
  | Database query           | Query time, row count, connection wait         |
  | Certificate generation   | Key gen time, signing time, total time         |
  | Cassandra operations     | Prepare time, execute time, result fetch       |
  | External service calls   | DNS lookup, connect time, response time        |
And calculate percentiles:
  | Percentile | Purpose                                            |
  | p50       | Median performance                                 |
  | p95       | Most users experience                              |
  | p99       | Worst case for most                               |
  | p999      | Extreme outliers                                   |
```

### Scenario: Alert Rule Configuration
```gherkin
Given metrics need to trigger alerts
When thresholds are exceeded
Then evaluate alert rules:
  | Alert                          | Condition                           | Severity | Action         |
  | High API Error Rate           | error_rate > 5% for 5m             | Critical | Page on-call   |
  | Certificate Expiry Soon       | days_until_expiry < 7              | Warning  | Email team     |
  | Database Connection Pool Full | connections_used/total > 0.9       | Warning  | Slack notify   |
  | Disk Space Low               | disk_free_percent < 10             | Critical | Page + Email   |
  | Cassandra Node Down          | node_status != "UN" for 2m         | Critical | Page on-call   |
  | Queue Backlog High           | queue_depth > 10000                | Warning  | Scale workers  |
  | Memory Usage High            | memory_percent > 85 for 10m        | Warning  | Investigate    |
And support alert routing:
  - Email for warnings
  - PagerDuty for critical
  - Slack for informational
  - Webhook for automation
```

### Scenario: Distributed Tracing
```gherkin
Given requests span multiple services
When tracing is enabled
Then implement OpenTelemetry spans:
  | Span                      | Attributes                                    |
  | HTTP Request             | method, path, status, user_id                 |
  | Database Query           | query_type, table, row_count                  |
  | Certificate Generation   | algorithm, key_size, duration                 |
  | External Service Call    | service, endpoint, status                     |
  | Background Task          | task_name, args, result                       |
And propagate context:
  - Extract trace headers from requests
  - Inject headers in outgoing calls
  - Link parent and child spans
  - Sample intelligently (1% normal, 100% errors)
```

### Scenario: Log Aggregation
```gherkin
Given logs need centralised analysis
When events are logged
Then use structured JSON format:
{
  "timestamp": "2024-01-15T10:30:45.123Z",
  "level": "INFO",
  "service": "cassandra-security-manager",
  "component": "certificate-service",
  "trace_id": "a1b2c3d4e5f6",
  "span_id": "1234567890",
  "user_id": "user-123",
  "message": "Certificate generated successfully",
  "attributes": {
    "certificate_id": "cert-456",
    "algorithm": "RSA-4096",
    "duration_ms": 1523
  }
}
And configure log levels:
  | Component          | Default Level | Configurable |
  | API               | INFO          | Yes          |
  | Security          | INFO          | Yes          |
  | Database          | WARN          | Yes          |
  | Cryptography      | ERROR         | No           |
  | Audit             | INFO          | No           |
```

### Scenario: Dashboard Integration
```gherkin
Given operators need visual monitoring
When metrics are collected
Then support dashboard systems:
  | System           | Integration Method         | Features                    |
  | Grafana         | Prometheus datasource      | Pre-built dashboards        |
  | Datadog         | StatsD + API              | APM integration             |
  | New Relic       | Agent + API               | Custom dashboards           |
  | CloudWatch      | AWS SDK                   | Native AWS integration      |
And provide dashboard templates for:
  - System overview (golden signals)
  - Certificate operations
  - Security events
  - Performance analysis
  - Capacity planning
```

### Scenario: SLA Monitoring
```gherkin
Given SLAs must be tracked
When calculating service levels
Then measure:
  | SLI                        | Target  | Measurement                        |
  | API Availability          | 99.9%   | Successful health checks           |
  | API Latency (p95)         | 200ms   | 95th percentile response time      |
  | Certificate Generation    | 99.5%   | Success rate over 5 minutes        |
  | Error Rate                | <0.1%   | 5xx errors / total requests        |
  | Certificate Availability  | 99.99%  | Successful retrievals              |
And calculate error budgets:
  - Monthly error budget = (1 - SLA) * minutes
  - Alert when 50% consumed
  - Page when 80% consumed
  - Freeze changes at 100%
```

## Edge Cases and Monitoring Considerations

### Edge Case: Monitoring System Failure
```gherkin
Given monitoring can fail
When Prometheus is unavailable
Then:
  - Continue exposing metrics endpoint
  - Buffer recent metrics in memory
  - Log metrics to backup location
  - Alert via alternative channel
  - Ensure core operations continue
```

### Edge Case: Metric Cardinality Explosion
```gherkin
Given too many label combinations cause issues
When high cardinality is detected
Then:
  - Limit label values (top 100)
  - Aggregate similar values
  - Alert on cardinality growth
  - Implement metric quotas
  - Document label guidelines
```

### Edge Case: Clock Skew
```gherkin
Given distributed systems may have clock differences
When timestamps are compared
Then:
  - Use monotonic clocks for durations
  - Include timezone in timestamps
  - Detect significant skew
  - Alert operators
  - Document NTP requirements
```

## Technical Notes
- Use Prometheus client library with proper labels
- Implement custom collectors for business metrics
- Use histograms for latency, summaries sparingly
- Keep cardinality under control (<10k series)
- Export metrics asynchronously
- Cache expensive metric calculations
- Use metric namespaces: `csm_api_*`, `csm_cert_*`
- Implement metric unit suffixes: `_seconds`, `_bytes`

## Definition of Done
- [ ] Health check endpoints implemented (/health, /health/live, /health/ready)
- [ ] Component health checks with appropriate thresholds
- [ ] Prometheus metrics endpoint at /metrics
- [ ] All business metrics instrumented
- [ ] Performance metrics with percentiles
- [ ] Alert rules configured and tested
- [ ] Distributed tracing implemented
- [ ] Structured JSON logging throughout
- [ ] Dashboard templates created
- [ ] SLA monitoring in place
- [ ] Monitoring runbook documented
- [ ] Load test metrics collection
- [ ] Metric cardinality controls
- [ ] Integration tests for health checks
- [ ] Documentation for all metrics