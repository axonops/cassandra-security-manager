# User Story 06.1 - Root CA Initialisation

## User Story
**As a** security administrator  
**I want** to establish a root Certificate Authority  
**So that** I can build a trusted certificate hierarchy with maximum security

## INVEST Criteria
- **Independent**: Root CA can be created without other features
- **Negotiable**: Specific algorithms and key sizes can be configured
- **Valuable**: Essential foundation for all certificate operations
- **Estimable**: 3 sprints including security controls
- **Small**: Focused on root CA creation only
- **Testable**: Certificate validation and security controls verifiable

## Acceptance Criteria

### Scenario: First-Time Root CA Generation
```gherkin
Given I am initialising the certificate infrastructure
When I POST to /ca/root/initialise with:
{
  "common_name": "Cassandra Security Manager Root CA",
  "organisation": "ACME Corp",
  "country": "US",
  "validity_years": 20,
  "algorithm": "RSA",
  "key_size": 4096,
  "digest": "SHA384"
}
Then the system should:
  | Step                      | Action                                        |
  | Check existing           | Verify no root CA exists                      |
  | Validate parameters      | Ensure algorithm and key size are approved    |
  | Generate key pair        | Create private/public key pair                |
  | Create certificate       | Self-signed root certificate                  |
  | Set extensions           | CA:TRUE, KeyUsage, pathLenConstraint         |
  | Store securely           | Encrypt private key with KEK                  |
  | Generate backups         | Create key shares for recovery                |
  | Audit log                | Record complete operation                     |
And return:
{
  "certificate_id": "uuid",
  "fingerprint": "SHA256:...",
  "subject_dn": "CN=...,O=...,C=...",
  "not_before": "2024-01-15T00:00:00Z",
  "not_after": "2044-01-15T23:59:59Z",
  "public_key": "-----BEGIN CERTIFICATE-----...",
  "backup_shares": ["share1", "share2", "share3", "share4", "share5"]
}
```

### Scenario: Quantum-Resistant Root CA
```gherkin
Given quantum computers pose future threats
When I create a root CA with post-quantum algorithms
Then support hybrid certificates:
  | Algorithm      | Purpose                | Key Size/Level    | Standard       |
  | CRYSTALS-Dilithium | Primary signing   | Level 3           | NIST PQC       |
  | RSA            | Fallback signing      | 4096 bits         | PKCS#1         |
  | Composite      | Combined signature    | Both              | RFC Draft      |
And the certificate should:
  - Include both public keys
  - Use composite signature format
  - Validate with either algorithm
  - Maintain compatibility with legacy systems
```

### Scenario: Air-Gap Security Enforcement
```gherkin
Given root CA must be air-gapped
When initialising root CA
Then enforce security measures:
  | Measure                   | Implementation                              |
  | Network isolation        | Detect and reject if network active         |
  | Offline mode             | All operations work without connectivity    |
  | Import/Export            | Via QR codes or USB with verification       |
  | Time synchronisation     | Manual time setting with drift detection    |
  | Operation logging        | Local tamper-evident logs                   |
And provide offline tools:
  - Standalone initialisation binary
  - Offline documentation
  - Verification utilities
  - Secure transfer protocols
```

### Scenario: Root CA Constraints and Extensions
```gherkin
Given root CA needs proper constraints
When generating the certificate
Then set critical extensions:
  | Extension                 | Value                    | Critical |
  | basicConstraints         | CA:TRUE, pathLen:2       | Yes      |
  | keyUsage                 | keyCertSign, cRLSign     | Yes      |
  | subjectKeyIdentifier     | Hash of public key       | No       |
  | authorityKeyIdentifier   | Self (same as SKI)       | No       |
  | certificatePolicies      | Organisation policies    | No       |
  | nameConstraints          | Permitted subtrees       | Yes      |
And enforce policy:
  - Maximum 2 levels of subordinate CAs
  - Only sign CA certificates, never end-entity
  - Restrict to organisation's domains
```

### Scenario: Key Ceremony Process
```gherkin
Given root CA creation requires ceremony
When performing key ceremony
Then follow formal process:
  | Phase                    | Requirements                                |
  | Preparation             | Clean room, tamper-evident hardware         |
  | Witnesses               | Minimum 3 authorised witnesses              |
  | Generation              | Entropy source verification                 |
  | Verification            | Independent key pair validation             |
  | Backup                  | Generate 5 shares, threshold 3              |
  | Distribution            | Secure courier to geographic locations      |
  | Destruction             | Secure wipe of generation system            |
  | Documentation           | Complete ceremony log with signatures       |
And record in blockchain or similar for non-repudiation
```

### Scenario: Hardware Security Module Integration
```gherkin
Given HSM provides superior key protection
When HSM is available
Then use for root CA:
  | Operation               | HSM Function                                |
  | Key generation         | Generate within HSM FIPS boundary           |
  | Key storage            | Non-extractable key storage                 |
  | Signing                | Sign within HSM                             |
  | Backup                 | HSM-to-HSM cloning only                     |
  | Access control         | M of N authentication                       |
And support HSM types:
  - Thales nShield
  - SafeNet Luna
  - AWS CloudHSM
  - Azure Dedicated HSM
  - PKCS#11 compatible devices
```

### Scenario: Emergency Recovery Procedures
```gherkin
Given disasters may require key recovery
When implementing recovery
Then provide mechanisms:
  | Scenario               | Recovery Method                              |
  | HSM failure           | Restore from HSM backup                      |
  | Key compromise        | Revoke and generate new root                 |
  | Share loss            | Reconstruct from remaining shares            |
  | Facility loss         | Geographic distribution of shares            |
  | Personnel loss        | Succession planning for share holders        |
And test recovery:
  - Annual recovery drills
  - Documented procedures
  - Secure communication channels
  - Legal framework for emergencies
```

### Scenario: Root CA Validation
```gherkin
Given root CA must be properly formed
When validation is performed
Then verify:
  | Check                    | Expected Result                             |
  | Self-signature          | Valid signature with public key             |
  | Key strength            | Meets minimum requirements                  |
  | Extensions              | All required extensions present             |
  | Validity period         | 20+ years remaining                         |
  | Algorithm               | Approved algorithm only                     |
  | Entropy                 | Sufficient randomness in key                |
  | Uniqueness              | No duplicate keys exist                     |
And perform crypto validation:
  - Generate test signature and verify
  - Ensure key pair correspondence
  - Check for weak key patterns
  - Verify prime number quality (RSA)
```

### Scenario: Compliance and Standards
```gherkin
Given regulatory requirements exist
When creating root CA
Then comply with standards:
  | Standard               | Requirements                                 |
  | FIPS 140-2 Level 3    | HSM validation for key storage              |
  | Common Criteria       | EAL4+ for CA systems                        |
  | WebTrust              | Audit procedures for public CAs             |
  | ETSI EN 319 411       | European qualified certificates             |
  | RFC 5280              | X.509 certificate profile                   |
And maintain evidence:
  - Ceremony videos and logs
  - Witness statements
  - Configuration records
  - Audit trails
```

## Edge Cases and Security Considerations

### Edge Case: Duplicate Root CA Attempt
```gherkin
Given a root CA already exists
When attempting to create another
Then:
  - Reject with clear error message
  - Log the attempt with full details
  - Alert security team
  - Require special override for replacement
  - Maintain previous root for verification
```

### Edge Case: Insufficient Entropy
```gherkin
Given key generation requires randomness
When entropy is insufficient
Then:
  - Detect low entropy conditions
  - Refuse to generate keys
  - Provide entropy gathering guidance
  - Support external entropy sources
  - Log entropy metrics
```

### Edge Case: Algorithm Becomes Compromised
```gherkin
Given cryptographic breaks happen
When an algorithm is compromised
Then:
  - Support rapid algorithm migration
  - Maintain old root for verification only
  - Generate new root with secure algorithm
  - Implement transition period
  - Update all dependent certificates
```

## Security Hardening

```gherkin
Given root CA is critical infrastructure
Then implement additional protections:
  | Protection              | Implementation                              |
  | Physical security      | Biometric access to HSM                     |
  | Dual control           | Two-person authorisation                    |
  | Time delays            | Enforced delays between operations          |
  | Rate limiting          | Maximum one root operation per day          |
  | Forensic logging       | Cryptographic proof of all operations       |
```

## Technical Notes
- Use cryptography.io library for key generation
- Implement PKCS#11 interface for HSM integration
- Support key generation in TEE (Trusted Execution Environment)
- Use Shamir's Secret Sharing for key backup
- Implement proper random number generation
- Consider post-quantum crypto libraries (liboqs)
- Plan for algorithm agility
- Support offline operation modes

## Definition of Done
- [ ] Root CA generation API implemented
- [ ] Quantum-resistant algorithms supported
- [ ] Air-gap security controls enforced
- [ ] Proper certificate extensions set
- [ ] Key ceremony process documented
- [ ] HSM integration implemented
- [ ] Emergency recovery tested
- [ ] Validation checks comprehensive
- [ ] Compliance requirements met
- [ ] Edge cases handled
- [ ] Security hardening applied
- [ ] Unit tests with mocked HSM
- [ ] Integration tests with SoftHSM
- [ ] Security audit completed
- [ ] Performance benchmarks documented
- [ ] Operational runbook created
- [ ] Training materials prepared