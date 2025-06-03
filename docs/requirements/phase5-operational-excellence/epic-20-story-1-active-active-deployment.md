# 20.1 - Active-Active Deployment

## User Story
**As a** platform reliability engineer  
**I want** active-active multi-region deployment  
**So that** the certificate management system achieves 99.99% availability with zero downtime

## INVEST Criteria
- **Independent**: Active-active deployment can be developed separately from single-region features
- **Negotiable**: HA architecture patterns and regions are configurable
- **Valuable**: Ensures business continuity and eliminates single points of failure
- **Estimable**: 5 sprints for comprehensive active-active implementation
- **Small**: Focused on high availability architecture only
- **Testable**: Availability metrics and failover scenarios are measurable

## Acceptance Criteria

### Scenario: Multi-Region Active-Active Architecture
```gherkin
Given the platform requires extreme availability
When deploying active-active architecture
Then implement comprehensive HA design:
{
  "active_active_architecture": {
    "deployment_topology": {
      "regions": {
        "primary": {
          "name": "us-east-1",
          "role": "active",
          "capacity": "100%",
          "components": ["api", "workers", "database", "cache", "hsm"]
        },
        "secondary": {
          "name": "us-west-2",
          "role": "active",
          "capacity": "100%",
          "components": ["api", "workers", "database", "cache", "hsm"]
        },
        "tertiary": {
          "name": "eu-central-1",
          "role": "active",
          "capacity": "100%",
          "components": ["api", "workers", "database", "cache", "hsm"]
        }
      },
      "availability_zones": {
        "per_region": 3,
        "distribution": "even_spread",
        "isolation": "power_and_network_independent"
      }
    },
    "data_architecture": {
      "database": {
        "technology": "cassandra",
        "replication_factor": 3,
        "consistency_level": "LOCAL_QUORUM",
        "cross_region_replication": "EACH_QUORUM"
      },
      "cache": {
        "technology": "redis_cluster",
        "replication": "master_slave_with_sentinel",
        "persistence": "aof_with_rdb_snapshots"
      },
      "message_queue": {
        "technology": "kafka",
        "replication_factor": 3,
        "min_in_sync_replicas": 2
      }
    },
    "network_architecture": {
      "load_balancing": {
        "global": "geoDNS_with_health_checks",
        "regional": "application_load_balancers",
        "internal": "service_mesh"
      },
      "connectivity": {
        "inter_region": "dedicated_network_backbone",
        "encryption": "ipsec_vpn_with_macsec",
        "bandwidth": "10gbps_minimum"
      }
    }
  }
}
And ensure characteristics:
  - No single point of failure
  - Geographic distribution
  - Automatic failover
  - Load distribution
  - Data consistency
```

### Scenario: Intelligent Traffic Management
```gherkin
Given traffic needs optimal routing
When managing request distribution
Then implement intelligent routing:
  | Traffic Feature      | Implementation                         |
  | Geographic routing  | Route to nearest healthy region        |
  | Load balancing      | Weighted round-robin with health       |
  | Failover routing    | Automatic rerouting on failure         |
  | Canary deployments  | Percentage-based traffic splitting     |
  | Circuit breaking    | Prevent cascade failures               |
And Optimise performance:
  - Latency-based routing
  - Connection pooling
  - Request coalescing
  - Adaptive rate limiting
  - Backpressure handling
```

### Scenario: Multi-Master Data Replication
```gherkin
Given data must be available in all regions
When implementing data replication
Then use multi-master patterns:
  | Data Type            | Replication Strategy                   |
  | Certificate data    | Synchronous multi-master               |
  | Configuration       | Eventual consistency with versioning   |
  | Audit logs          | Async replication with ordering        |
  | Session data        | Sticky sessions with replication       |
  | Metrics             | Local collection, global aggregation   |
And handle conflicts:
  - Last-write-wins for configs
  - Vector clocks for complex data
  - Conflict-free replicated data types
  - Manual resolution workflows
  - Conflict monitoring
```

### Scenario: Automated Health Monitoring
```gherkin
Given system health determines routing
When monitoring component health
Then implement comprehensive checks:
  | Health Check Type    | Frequency and Timeout                  |
  | API endpoints       | Every 5s, 2s timeout                   |
  | Database connections| Every 10s, 5s timeout                  |
  | HSM availability    | Every 30s, 10s timeout                 |
  | Certificate operations| Every 60s, 30s timeout               |
  | Replication lag     | Continuous monitoring                  |
And calculate health scores:
  - Component-level health (0-100)
  - Service-level health aggregation
  - Regional health scores
  - Global health dashboard
  - Predictive health analytics
```

### Scenario: Zero-Downtime Failover
```gherkin
Given failures must not impact users
When failover occurs
Then ensure zero downtime:
  | Failover Scenario    | Recovery Actions                       |
  | Region failure      | Redirect traffic to healthy regions    |
  | AZ failure          | Rebalance within region                |
  | Component failure   | Use local redundancy                   |
  | Network partition   | Maintain quorum operations             |
  | Database failure    | Promote replica automatically          |
And maintain state:
  - Session preservation
  - Transaction completion
  - Queue durability
  - Cache warming
  - Connection draining
```

### Scenario: Disaster Recovery Automation
```gherkin
Given disasters require rapid response
When disaster strikes
Then automate recovery:
  | Disaster Type        | Automated Response                     |
  | Region loss         | Activate DR region                     |
  | Data corruption     | Point-in-time recovery                 |
  | Cyber attack        | Isolate and restore                    |
  | Natural disaster    | Pre-positioned capacity activation     |
  | Human error         | Automated rollback                     |
And ensure RPO/RTO:
  - RPO: 0 seconds (synchronous replication)
  - RTO: < 5 minutes (automated activation)
  - Continuous backup verification
  - Regular DR drills
  - Runbook automation
```

### Scenario: Consensus and Coordination
```gherkin
Given distributed systems need coordination
When implementing consensus
Then use proven algorithms:
  | Consensus Need       | Implementation                         |
  | Leader election     | Raft consensus algorithm               |
  | Configuration mgmt  | etcd/Consul with watches              |
  | Lock management     | Distributed locks with TTL             |
  | Queue coordination  | Kafka consumer groups                  |
  | Schema management   | Versioned migrations with locks        |
And prevent split-brain:
  - Quorum-based decisions
  - Fencing tokens
  - Network partition detection
  - Automatic leader step-down
  - Manual override capabilities
```

### Scenario: Performance Under Failure
```gherkin
Given performance must be maintained during failures
When components fail
Then maintain SLAs:
  | Failure Mode         | Performance Target                     |
  | Single region down  | No performance impact                  |
  | 50% capacity loss   | < 20% latency increase                |
  | Database failover   | < 30 second interruption              |
  | Network degradation | Graceful degradation                   |
  | Cascading failures  | Circuit breakers prevent spread        |
And implement:
  - Auto-scaling on failure
  - Request hedging
  - Speculative execution
  - Timeout tuning
  - Load shedding
```

### Scenario: Operational Runbooks
```gherkin
Given operations need Standardised procedures
When creating runbooks
Then automate operational tasks:
  | Runbook Type         | Automation Level                       |
  | Failover procedures | Fully automated with approval          |
  | Capacity scaling    | Auto-scale with limits                 |
  | Backup verification | Scheduled automated tests              |
  | Security patching   | Rolling updates with validation        |
  | Incident response   | Automated triage and escalation        |
And provide:
  - Step-by-step procedures
  - Automated execution options
  - Rollback procedures
  - Success criteria
  - Post-mortem templates
```

### Scenario: Chaos Engineering
```gherkin
Given systems must be resilient
When testing resilience
Then implement chaos engineering:
  | Chaos Experiment     | Testing Approach                       |
  | Random pod kills    | Kill random services hourly            |
  | Network latency     | Inject 100-500ms latency               |
  | Region failure      | Simulate full region outage            |
  | Data corruption     | Corrupt non-critical data              |
  | Resource exhaustion | CPU/memory/disk pressure               |
And measure:
  - Mean time to detection
  - Mean time to recovery
  - Customer impact metrics
  - Automation effectiveness
  - Improvement opportunities
```

## Edge Cases and Security Considerations

### Edge Case: Simultaneous Multi-Region Failures
```gherkin
Given multiple regions may fail together
When handling multi-region failures
Then:
  - Maintain quorum in remaining regions
  - Activate emergency capacity
  - Implement degraded service modes
  - Prioritize critical operations
  - Communicate status clearly
```

### Edge Case: Data Consistency During Partition
```gherkin
Given network partitions affect consistency
When partitions occur
Then:
  - Detect partition quickly
  - Maintain consistency guarantees
  - Queue writes if needed
  - Reconcile when healed
  - Alert on inconsistencies
```

### Edge Case: Cascading Failures
```gherkin
Given failures can cascade
When detecting cascade risk
Then:
  - Implement circuit breakers
  - Use bulkheads for isolation
  - Apply backpressure
  - Shed non-critical load
  - Monitor dependency health
```

### Edge Case: Byzantine Failures
```gherkin
Given components may fail arbitrarily
When Byzantine failures occur
Then:
  - Use Byzantine fault-tolerant consensus
  - Implement voting mechanisms
  - Verify state transitions
  - Detect misbehaving nodes
  - Isolate faulty components
```

## Security Hardening

```gherkin
Given HA systems expand attack surface
Then implement comprehensive security:
  | Security Control     | Implementation                         |
  | Network security    | Zero-trust between regions             |
  | Encryption          | TLS 1.3 for all inter-region traffic   |
  | Authentication      | mTLS between services                  |
  | Access control      | Region-specific service accounts       |
  | Secret management   | Regional secret replication            |
  | DDoS protection     | Regional scrubbing Centres             |
  | Intrusion detection | Distributed IDS/IPS                    |
  | Security monitoring | Global SOC with regional presence      |
```

## Performance Requirements

```gherkin
Given HA must not compromise performance
Then meet these targets:
  | Performance Metric   | Target                                 |
  | Availability        | 99.99% (52.56 minutes downtime/year)  |
  | Failover time       | < 30 seconds automatic failover        |
  | Cross-region latency| < 100ms replication lag               |
  | API response time   | < 200ms p99 globally                   |
  | Throughput          | 100,000+ operations/second globally    |
  | Data durability     | 99.999999999% (11 nines)              |
```

## HA Configuration Examples

### Global Load Balancer Configuration
```yaml
global_load_balancer:
  type: "geo_dns"
  
  health_checks:
    interval: 5
    timeout: 2
    unhealthy_threshold: 2
    healthy_threshold: 3
    
  routing_policy:
    type: "weighted_latency"
    weights:
      us_east_1: 33
      us_west_2: 33
      eu_central_1: 34
      
  failover:
    detection_time: 10
    failover_threshold: "50%_unhealthy"
    
  endpoints:
    - region: "us-east-1"
      endpoint: "us-east-1.api.certmanager.com"
      priority: 1
      weight: 100
      
    - region: "us-west-2"
      endpoint: "us-west-2.api.certmanager.com"
      priority: 1
      weight: 100
      
    - region: "eu-central-1"
      endpoint: "eu-central-1.api.certmanager.com"
      priority: 1
      weight: 100
```

### Cassandra Multi-Region Configuration
```yaml
cassandra:
  cluster_name: "cert_manager_global"
  
  datacenters:
    us_east_1:
      replication_factor: 3
      nodes: ["cass-use1-1", "cass-use1-2", "cass-use1-3"]
      
    us_west_2:
      replication_factor: 3
      nodes: ["cass-usw2-1", "cass-usw2-2", "cass-usw2-3"]
      
    eu_central_1:
      replication_factor: 3
      nodes: ["cass-euc1-1", "cass-euc1-2", "cass-euc1-3"]
      
  keyspace_replication:
    strategy: "NetworkTopologyStrategy"
    us_east_1: 3
    us_west_2: 3
    eu_central_1: 3
    
  consistency_levels:
    read: "LOCAL_QUORUM"
    write: "EACH_QUORUM"
    
  cross_dc_latency_aware: true
  prefer_local: true
```

### Chaos Engineering Test
```python
@chaos_test
def test_region_failure_resilience():
    """Test system resilience to complete region failure"""
    
    # Baseline metrics
    baseline_availability = measure_availability()
    baseline_latency = measure_latency()
    
    # Simulate region failure
    regions = ["us-east-1", "us-west-2", "eu-central-1"]
    failed_region = random.choice(regions)
    
    with simulate_region_failure(failed_region):
        # Measure during failure
        failure_availability = measure_availability()
        failure_latency = measure_latency()
        failover_time = measure_failover_time()
        
        # Verify SLAs maintained
        assert failure_availability >= 0.99
        assert failure_latency < baseline_latency * 1.5
        assert failover_time < 30  # seconds
        
        # Test operations continue
        test_certificate_operations()
        test_data_consistency()
        
    # Verify recovery
    recovery_metrics = measure_recovery()
    assert recovery_metrics.data_loss == 0
    assert recovery_metrics.inconsistencies == 0
```

## Technical Notes
- Use Kubernetes operators for automated orchestration
- Implement service mesh for inter-service communication
- Use distributed tracing for failure analysis
- Plan for 3x capacity for N+2 redundancy
- Implement automated capacity planning
- Use anycast for global load balancing
- Design for eventual consistency where appropriate
- Monitor cross-region bandwidth costs

## Definition of Done
- [ ] Multi-region active-active deployed
- [ ] Intelligent traffic management operational
- [ ] Multi-master replication functional
- [ ] Automated health monitoring active
- [ ] Zero-downtime failover tested
- [ ] Disaster recovery automation complete
- [ ] Consensus mechanisms implemented
- [ ] Performance under failure validated
- [ ] Operational runbooks automated
- [ ] Chaos engineering framework active
- [ ] Edge cases handled
- [ ] Security hardening applied
- [ ] Performance targets met
- [ ] 99.99% availability achieved
- [ ] Documentation comprehensive