# User Story 06.3 - Subordinate CA Generation

## User Story
**As a** security administrator  
**I want** to create subordinate CAs  
**So that** I can implement proper certificate segregation and delegation

## INVEST Criteria
- **Independent**: Can be developed after root CA exists
- **Negotiable**: Number and structure of subordinate CAs flexible
- **Valuable**: Enables operational certificate signing
- **Estimable**: 2 sprints including validation
- **Small**: Focused on subordinate CA creation
- **Testable**: Chain validation and constraints verifiable

## Acceptance Criteria

### Scenario: Cassandra-Specific CA Creation
```gherkin
Given an established root CA exists
When I POST to /ca/subordinate/create with:
{
  "common_name": "Cassandra Cluster CA",
  "parent_ca_id": "root-ca-uuid",
  "purpose": "CASSANDRA_NODES",
  "validity_years": 10,
  "algorithm": "ECDSA",
  "curve": "P-384",
  "path_length": 1,
  "name_constraints": {
    "permitted": ["*.cassandra.local", "*.node.cluster"]
  }
}
Then the system should:
  | Step                    | Action                                         |
  | Validate parent        | Ensure root CA exists and is valid            |
  | Check authorisation    | Verify user can create subordinate CAs         |
  | Generate key pair      | Create new key pair for subordinate           |
  | Create CSR             | Certificate signing request with constraints   |
  | Sign with root         | Use root CA to sign (may be offline)          |
  | Verify chain           | Ensure proper chain to root                    |
  | Store certificate      | Save with relationship to parent               |
  | Update CA registry     | Mark as available for signing                  |
And return subordinate CA certificate with proper constraints
```

### Scenario: Offline Root CA Signing
```gherkin
Given root CA operates in air-gapped mode
When subordinate CA needs signing
Then support offline workflow:
  | Phase                  | Process                                        |
  | CSR Generation        | Create CSR with all extensions                 |
  | Export Request        | Generate QR code or save to USB                |
  | Transfer              | Secure physical transfer to root CA system     |
  | Offline Signing       | Sign on air-gapped system                      |
  | Import Certificate    | Scan QR or load from USB                       |
  | Verification          | Validate signature and chain                   |
  | Activation            | Enable subordinate CA for operations           |
And maintain security:
  - Tamper-evident transfer media
  - Cryptographic verification codes
  - Audit trail for physical transfer
  - Time-bound validity for requests
```

### Scenario: Purpose-Specific Subordinate CAs
```gherkin
Given different certificate types need isolation
When creating subordinate CAs
Then enforce purpose-based separation:
  | CA Purpose           | Allowed Certificates      | Constraints                |
  | Node CA             | Cassandra node certs      | CN must match node pattern |
  | Client CA           | Application certs         | Client auth EKU only       |
  | User CA             | Individual user certs     | Email in SAN required      |
  | Service CA          | Microservice certs        | Specific DNS names only    |
  | Monitoring CA       | Monitoring tool certs     | Short validity (90 days)   |
And implement constraints:
  - Extended Key Usage restrictions
  - Name constraints for domains
  - Path length limitations
  - Certificate policies
```

### Scenario: Hierarchical CA Structure
```gherkin
Given complex organisations need hierarchy
When building CA structure
Then support multiple levels:
```
Root CA (Offline)
├── Infrastructure CA (Level 1)
│   ├── Cassandra CA (Level 2)
│   │   ├── Production Cluster CA
│   │   └── Development Cluster CA
│   └── Application CA (Level 2)
│       ├── Internal Services CA
│       └── External Services CA
└── User CA (Level 1)
    ├── Employee CA (Level 2)
    └── Contractor CA (Level 2)
```
And enforce path length:
  - Root: pathLen=2
  - Level 1: pathLen=1
  - Level 2: pathLen=0 (no further CAs)
```

### Scenario: CA Certificate Extensions
```gherkin
Given subordinate CAs need proper marking
When generating certificates
Then set appropriate extensions:
  | Extension              | Value                          | Purpose                    |
  | basicConstraints      | CA:TRUE, pathLen:n             | Mark as CA with depth      |
  | keyUsage              | keyCertSign, digitalSignature  | Signing permissions        |
  | extKeyUsage           | Per purpose (serverAuth, etc)  | Restrict usage             |
  | certificatePolicies   | Organisation OIDs              | Policy compliance          |
  | crlDistributionPoints | https://crl.example.com/ca    | Revocation checking        |
  | authorityInfoAccess   | OCSP and CA endpoints          | Chain discovery            |
  | nameConstraints       | Permitted/excluded subtrees    | Domain restrictions        |
And validate all extensions are properly critical/non-critical
```

### Scenario: Cross-Certification Support
```gherkin
Given migration scenarios require trust bridging
When implementing cross-certification
Then support:
  | Scenario              | Implementation                                 |
  | Old to new root      | Cross-sign new root with old                   |
  | External trust       | Cross-certify partner CAs                      |
  | Algorithm migration  | Bridge RSA to ECC hierarchies                  |
  | Merger/acquisition   | Unite separate CA structures                   |
And maintain:
  - Clear validity periods
  - Audit trails for trust decisions
  - Revocation capabilities
  - Policy mapping
```

### Scenario: Subordinate CA Validation
```gherkin
Given subordinate CAs must be properly formed
When validating a subordinate CA
Then verify:
  | Check                 | Validation                                     |
  | Parent signature     | Valid signature from parent CA                 |
  | Chain integrity      | Complete chain to root CA                      |
  | Constraints          | Not exceeding parent constraints               |
  | Extensions           | All required extensions present                |
  | Validity period      | Within parent validity period                  |
  | Key strength         | Meets minimum requirements                     |
  | Purpose alignment    | Matches intended use case                      |
And reject if any validation fails
```

### Scenario: CA Inventory Management
```gherkin
Given multiple CAs need tracking
When querying CA inventory
Then provide comprehensive view:
  | Information          | Details                                        |
  | CA hierarchy        | Visual tree representation                     |
  | Certificate details | Subject, validity, algorithms                  |
  | Usage statistics    | Certificates issued, active, revoked           |
  | Health status       | Expiry warnings, key strength                  |
  | Constraints         | Effective constraints from hierarchy           |
And support operations:
  - Export CA certificates
  - Download certificate chains
  - Monitor expiration
  - Plan renewals
```

### Scenario: Automated CA Renewal
```gherkin
Given CAs expire before end-entity certificates
When CA approaches expiration
Then automate renewal:
  | Time Before Expiry   | Action                                        |
  | 1 year              | Alert administrators                           |
  | 6 months            | Generate renewal plan                          |
  | 3 months            | Create new CA with same key                    |
  | 1 month             | Begin transitioning certificates               |
  | 1 week              | Final warning, disable old CA                  |
And maintain service:
  - Overlap validity periods
  - Dual-sign during transition
  - Update all dependent systems
  - Verify no orphaned certificates
```

## Edge Cases and Security Considerations

### Edge Case: Parent CA Compromise
```gherkin
Given a parent CA may be compromised
When compromise is detected
Then:
  - Immediately revoke parent CA
  - Mark all subordinate CAs as untrusted
  - Generate new hierarchy
  - Implement emergency rotation
  - Notify all certificate users
  - Maintain revocation information
```

### Edge Case: Constraint Violations
```gherkin
Given constraints prevent certain operations
When constraint violation is attempted
Then:
  - Clearly explain which constraint failed
  - Suggest alternative approaches
  - Log attempted violation
  - Alert security team for patterns
  - Provide override mechanism with audit
```

### Edge Case: CA Key Rotation
```gherkin
Given CA keys may need rotation
When rotating a CA key
Then:
  - Generate new key pair
  - Create new certificate with same DN
  - Maintain old key for verification
  - Issue new certificates with new CA
  - Plan transition period
  - Update all trust stores
```

## Security Hardening

```gherkin
Given subordinate CAs are attack targets
Then implement protections:
  | Protection           | Implementation                                |
  | Access control      | Role-based CA creation                        |
  | Rate limiting       | Maximum CAs per hour/day                      |
  | Approval workflow   | Multi-person approval for CA creation         |
  | Audit logging       | Every CA operation logged                     |
  | Monitoring          | Alert on unusual CA operations                |
  | Backup              | Automated encrypted backups                   |
```

## Technical Notes
- Implement proper certificate chain building
- Support both RSA and ECDSA algorithms
- Use cryptography.io for certificate operations
- Implement PKCS#10 for CSR handling
- Support name constraints properly
- Cache CA certificates for performance
- Implement efficient chain validation
- Plan for CA certificate distribution

## Definition of Done
- [ ] Subordinate CA creation API implemented
- [ ] Offline root CA signing supported
- [ ] Purpose-specific CA types defined
- [ ] Hierarchical structure enforced
- [ ] All extensions properly set
- [ ] Cross-certification supported
- [ ] Validation comprehensive
- [ ] CA inventory management complete
- [ ] Automated renewal implemented
- [ ] Edge cases handled
- [ ] Security hardening applied
- [ ] Unit tests complete
- [ ] Integration tests with chains
- [ ] Performance tests for chain building
- [ ] Documentation complete
- [ ] Operational procedures defined