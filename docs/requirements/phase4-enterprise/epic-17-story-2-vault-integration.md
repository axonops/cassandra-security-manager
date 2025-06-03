# 17.2 - HashiCorp Vault Integration

## User Story
**As a** DevOps engineer  
**I want** HashiCorp Vault integration for Centralised secret management  
**So that** certificates and keys are managed through our enterprise secret management platform

## INVEST Criteria
- **Independent**: Vault integration can be developed separately from HSM features
- **Negotiable**: Integration depth and features are configurable
- **Valuable**: Enables Centralised secret management and dynamic credentials
- **Estimable**: 3 sprints for comprehensive Vault integration
- **Small**: Focused on Vault secret management integration only
- **Testable**: Integration functionality and performance are measurable

## Acceptance Criteria

### Scenario: Comprehensive Vault Backend Support
```gherkin
Given Vault supports multiple secret backends
When integrating with Vault
Then support key Vault engines:
{
  "vault_engines": {
    "pki_engine": {
      "path": "/pki",
      "features": [
        "root_ca_generation",
        "intermediate_ca_signing",
        "certificate_generation",
        "crl_management",
        "ocsp_responder"
      ],
      "configuration": {
        "max_ttl": "87600h",
        "default_ttl": "8760h",
        "crl_distribution": ["http://crl.example.com"],
        "ocsp_servers": ["http://ocsp.example.com"]
      }
    },
    "kv_v2_engine": {
      "path": "/secret",
      "features": [
        "versioned_secrets",
        "soft_delete",
        "secret_metadata",
        "cas_enforcement"
      ],
      "use_cases": [
        "api_keys",
        "database_credentials",
        "encryption_keys",
        "configuration_data"
      ]
    },
    "transit_engine": {
      "path": "/transit",
      "features": [
        "encryption_as_service",
        "key_rotation",
        "convergent_encryption",
        "key_derivation"
      ],
      "algorithms": [
        "aes256-gcm96",
        "chacha20-poly1305",
        "ed25519",
        "ecdsa-p384"
      ]
    },
    "database_engine": {
      "path": "/database",
      "features": [
        "dynamic_credentials",
        "credential_rotation",
        "lease_management"
      ],
      "supported_databases": [
        "cassandra",
        "postgresql",
        "mysql",
        "mongodb"
      ]
    }
  }
}
And implement engine integration:
  - Native API usage for each engine
  - Optimal configuration per engine
  - Error handling per engine type
  - Performance Optimisation
  - Feature detection and adaptation
```

### Scenario: Dynamic Certificate Management
```gherkin
Given certificates can be dynamically generated
When using Vault PKI engine
Then implement dynamic certificate management:
  | Certificate Type     | Vault Integration                      |
  | Root CA             | Import existing or generate in Vault   |
  | Intermediate CA     | Generate and sign via Vault            |
  | Server certificates | Dynamic generation with TTL            |
  | Client certificates | On-demand generation                   |
  | Code signing certs  | Policy-controlled generation           |
And manage certificate lifecycle:
  - Automatic renewal before expiry
  - Certificate revocation lists
  - OCSP responder integration
  - Certificate role templates
  - Usage auditing
```

### Scenario: Secret Storage and Retrieval
```gherkin
Given secrets need Centralised management
When storing secrets in Vault
Then implement secure storage:
  | Secret Type          | Storage Strategy                       |
  | Private keys        | Store key references, not keys         |
  | API credentials     | Versioned KV storage                   |
  | Database passwords  | Dynamic generation                     |
  | Encryption keys     | Transit engine management              |
  | Configuration       | Templated configuration                |
And provide retrieval features:
  - Caching with TTL respect
  - Automatic lease renewal
  - Secret rotation handling
  - Batch secret operations
  - Emergency access procedures
```

### Scenario: Vault Authentication Methods
```gherkin
Given different authentication methods exist
When authenticating to Vault
Then support multiple auth methods:
  | Auth Method          | Use Case                               |
  | AppRole             | Application authentication             |
  | Kubernetes          | Pod-based authentication               |
  | LDAP                | User authentication                    |
  | JWT/OIDC            | Token-based authentication             |
  | TLS Certificates    | Mutual TLS authentication              |
And implement auth features:
  - Automatic token renewal
  - Auth method fallback
  - Token caching strategies
  - Auth audit logging
  - Emergency auth procedures
```

### Scenario: Policy-Based Access Control
```gherkin
Given Vault uses policy-based access
When managing access control
Then implement policy management:
  | Policy Component     | Implementation                         |
  | Path policies       | Fine-grained path access               |
  | Capability policies | Read, write, delete, list permissions  |
  | Template policies   | Dynamic policy generation              |
  | Sentinel policies   | Advanced policy as code                |
  | Identity policies   | Entity and group based policies        |
And enforce policies:
  - Policy Synchronisation
  - Policy testing framework
  - Policy violation handling
  - Audit policy usage
  - Policy version control
```

### Scenario: High Availability and Performance
```gherkin
Given Vault must be highly available
When implementing Vault integration
Then ensure HA operations:
  | HA Feature           | Implementation                         |
  | Cluster support     | Multi-node Vault cluster connection    |
  | Load balancing      | Client-side load balancing             |
  | Failover handling   | Automatic leader discovery             |
  | Connection pooling  | Persistent connection management       |
  | Request retry       | Exponential backoff retry              |
And Optimise performance:
  - Response caching
  - Batch operations
  - Async secret retrieval
  - Connection multiplexing
  - Performance monitoring
```

### Scenario: Vault Namespace Integration
```gherkin
Given enterprises use Vault namespaces
When supporting multi-tenancy
Then implement namespace support:
  | Namespace Feature    | Implementation                         |
  | Tenant isolation    | Namespace per tenant                   |
  | Cross-namespace     | Controlled secret sharing              |
  | Namespace policies  | Inherited and local policies           |
  | Resource quotas     | Per-namespace limits                   |
  | Audit segregation   | Namespace-specific audit logs          |
And manage namespaces:
  - Automatic namespace provisioning
  - Namespace lifecycle management
  - Cross-namespace operations
  - Namespace migration
  - Disaster recovery per namespace
```

### Scenario: Secret Rotation and Renewal
```gherkin
Given secrets must be rotated regularly
When implementing rotation
Then automate secret lifecycle:
  | Rotation Type        | Implementation                         |
  | Automatic rotation  | Policy-based automatic rotation        |
  | Manual rotation     | On-demand rotation triggers            |
  | Lease renewal       | Automatic lease extension              |
  | Grace periods       | Overlapping secret validity            |
  | Rollback support    | Previous version access                |
And handle rotation events:
  - Pre-rotation notifications
  - Rotation success callbacks
  - Failure handling and retry
  - Zero-downtime rotation
  - Audit trail maintenance
```

### Scenario: Vault Audit Integration
```gherkin
Given all Vault operations must be audited
When integrating audit logs
Then implement audit features:
  | Audit Feature        | Implementation                         |
  | Log streaming       | Real-time audit log streaming          |
  | Log parsing         | Structured audit log parsing           |
  | Event correlation   | Correlate with application events      |
  | Compliance mapping  | Map to compliance requirements         |
  | Alert generation    | Security alerts from audit events      |
And process audit data:
  - Audit log enrichment
  - Long-term retention
  - Audit analytics
  - Compliance reporting
  - Forensic investigation support
```

### Scenario: Disaster Recovery Integration
```gherkin
Given Vault DR is critical
When implementing DR procedures
Then support Vault DR features:
  | DR Component         | Integration                            |
  | Backup integration  | Automated Vault backup                 |
  | Snapshot support    | Point-in-time recovery                 |
  | Replication        | DR replication cluster support         |
  | Failover           | Automated DR failover                  |
  | Data restoration   | Selective secret restoration           |
And ensure recovery:
  - DR testing procedures
  - Recovery time objectives
  - Cross-region DR support
  - Data consistency validation
  - DR audit compliance
```

## Edge Cases and Security Considerations

### Edge Case: Vault Seal/Unseal Events
```gherkin
Given Vault may be sealed
When Vault seal events occur
Then:
  - Detect seal status quickly
  - Queue operations during seal
  - Alert operations team
  - Prevent secret exposure
  - Implement graceful degradation
```

### Edge Case: Lease Expiration
```gherkin
Given Vault leases expire
When lease expiration occurs
Then:
  - Proactive lease renewal
  - Grace period handling
  - Fallback credentials
  - Alert on renewal failure
  - Maintain service availability
```

### Edge Case: Policy Conflicts
```gherkin
Given policies may conflict
When policy conflicts occur
Then:
  - Detect policy conflicts
  - Apply most restrictive policy
  - Log conflict details
  - Alert security team
  - Provide conflict resolution
```

### Edge Case: Network Partitions
```gherkin
Given network partitions may occur
When Vault cluster is partitioned
Then:
  - Detect partition state
  - Route to available nodes
  - Cache critical secrets
  - Implement split-brain handling
  - Maintain audit continuity
```

## Security Hardening

```gherkin
Given Vault integration requires security
Then implement comprehensive security:
  | Security Control     | Implementation                         |
  | TLS communication   | Mutual TLS with certificate pinning    |
  | Token security      | Short-lived tokens with auto-renewal   |
  | Secret protection   | In-memory only, no disk persistence    |
  | Audit security      | Encrypted audit log transmission       |
  | Access control      | Principle of least privilege           |
  | Network security    | Vault-specific network segments        |
  | Key derivation      | Unique keys per application            |
  | Emergency access    | Break-glass procedures with audit      |
```

## Performance Requirements

```gherkin
Given Vault operations must be performant
Then meet these performance targets:
  | Performance Metric   | Target                                 |
  | Secret retrieval    | < 50ms for cached secrets              |
  | Certificate generation| < 2 seconds for new certificates      |
  | Bulk operations     | > 1000 secrets per second              |
  | Token renewal       | < 100ms token refresh                  |
  | Failover time       | < 5 seconds to standby                 |
  | Cache hit rate      | > 90% for frequently used secrets      |
```

## Vault Integration Examples

### Vault Configuration
```yaml
vault_integration:
  address: "https://vault.example.com:8200"
  namespace: "cassandra-cert-manager"
  
  authentication:
    method: "approle"
    role_id: "${VAULT_ROLE_ID}"
    secret_id: "${VAULT_SECRET_ID}"
    
  tls:
    ca_cert: "/etc/vault/ca.pem"
    client_cert: "/etc/vault/client.pem"
    client_key: "/etc/vault/client-key.pem"
    tls_server_name: "vault.example.com"
    
  engines:
    pki:
      mount_path: "pki"
      roles:
        server: "server-cert"
        client: "client-cert"
      ttl: "720h"
      
    kv:
      mount_path: "secret"
      version: 2
      
    transit:
      mount_path: "transit"
      key_name: "cassandra-encryption"
      
  performance:
    max_retries: 3
    retry_wait_min: "1s"
    retry_wait_max: "30s"
    connection_timeout: "30s"
    max_idle_connections: 10
    
  caching:
    enable: true
    ttl: "5m"
    max_entries: 10000
```

### Certificate Generation via Vault
```json
{
  "request": {
    "path": "pki/issue/server-cert",
    "data": {
      "common_name": "cassandra-node-1.example.com",
      "alt_names": [
        "cassandra-node-1",
        "10.0.1.100"
      ],
      "ttl": "720h",
      "format": "pem",
      "private_key_format": "der",
      "exclude_cn_from_sans": false
    }
  },
  "response": {
    "certificate": "-----BEGIN CERTIFICATE-----...",
    "issuing_ca": "-----BEGIN CERTIFICATE-----...",
    "ca_chain": ["-----BEGIN CERTIFICATE-----..."],
    "private_key": "-----BEGIN RSA PRIVATE KEY-----...",
    "serial_number": "39:dd:2e:90:b7:23:1f:8d:d3:7d:31:c5:1b:da:84:d0:5b:65:31:58",
    "expiration": 1924905600
  }
}
```

### Secret Storage Example
```python
# Store database credentials
vault_client.secrets.kv.v2.create_or_update_secret(
    path="cassandra/prod/credentials",
    secret={
        "username": "cassandra_app",
        "password": generate_secure_password(),
        "connection_string": "cassandra://cluster.example.com:9042",
        "client_certificate": base64_encoded_cert,
        "client_key": base64_encoded_key
    },
    mount_point="secret"
)

# Retrieve with caching
@cached(ttl=300)
def get_cassandra_credentials():
    response = vault_client.secrets.kv.v2.read_secret_version(
        path="cassandra/prod/credentials",
        mount_point="secret"
    )
    return response["data"]["data"]
```

## Technical Notes
- Use Vault Agent for token management
- Implement circuit breakers for Vault calls
- Cache immutable data aggressively
- Use Vault batch APIs where available
- Plan for Vault Enterprise features
- Implement comprehensive retry logic
- Monitor Vault performance metrics
- Design for zero-trust architecture

## Definition of Done
- [ ] Comprehensive Vault backend support
- [ ] Dynamic certificate management operational
- [ ] Secret storage and retrieval implemented
- [ ] Multiple auth methods supported
- [ ] Policy-based access control working
- [ ] High availability configuration complete
- [ ] Namespace integration functional
- [ ] Secret rotation automated
- [ ] Audit integration operational
- [ ] Disaster recovery procedures tested
- [ ] Edge cases handled
- [ ] Security hardening applied
- [ ] Performance targets met
- [ ] DevOps team acceptance testing
- [ ] Documentation comprehensive