# User Story 04.2 - Secrets Management Foundation

## User Story
**As a** security administrator  
**I want** secure secrets storage  
**So that** sensitive cryptographic materials are protected with defence-in-depth

## INVEST Criteria
- **Independent**: Can be developed alongside other security controls
- **Negotiable**: Specific encryption algorithms can be adjusted
- **Valuable**: Essential for protecting private keys and credentials
- **Estimable**: 2 sprints including HSM preparation
- **Small**: Focused on secrets storage infrastructure
- **Testable**: Cryptographic operations can be verified

## Acceptance Criteria

### Scenario: Master Key Initialisation
```gherkin
Given the system needs a Key Encryption Key (KEK)
When the application starts for the first time
Then the master key initialisation should:
  | Step                          | Action                                      |
  | Generate master key          | Create 256-bit random key                   |
  | Protect master key           | Use PBKDF2 with 100,000 iterations        |
  | Store securely               | Environment variable or HSM                 |
  | Backup safely                | Shamir's Secret Sharing (3 of 5)          |
  | Rotate periodically          | Support key rotation without downtime       |
And the master key must never be:
  - Stored in the database
  - Logged in any form
  - Transmitted over network
  - Accessible via API
```

### Scenario: Private Key Encryption
```gherkin
Given a private key needs to be stored
When the key is persisted
Then the encryption process must:
  | Operation                    | Specification                               |
  | Generate DEK                | AES-256 key per private key                |
  | Encrypt private key         | AES-256-GCM with unique IV                 |
  | Encrypt DEK                 | Using KEK with AES-256-GCM                 |
  | Store metadata              | Algorithm, IV, auth tag, key ID            |
  | Generate key ID             | UUID v4 for reference                      |
And maintain the following properties:
  - Each private key has unique DEK
  - IVs are never reused
  - Authentication tags prevent tampering
  - Original key material is zeroed from memory
```

### Scenario: Secret Categorisation and Policies
```gherkin
Given different secrets have different security requirements
When storing secrets
Then apply category-specific policies:
  | Secret Type            | Encryption | Rotation | Access Control        | Audit |
  | Root CA Private Key   | HSM/KEK    | Never    | 2-person rule        | All   |
  | Intermediate CA Key   | KEK        | 5 years  | Admin only           | All   |
  | Node Certificate Key  | KEK        | 1 year   | System + Admin       | All   |
  | User Password Hash    | Argon2     | 90 days  | User + Admin         | Login |
  | API Token            | HMAC       | 180 days | Owner only           | All   |
  | Database Password    | KEK        | 90 days  | System only          | Start |
And enforce mandatory rotation schedules
```

### Scenario: Secret Retrieval and Decryption
```gherkin
Given an Authorised request for a secret
When retrieving the secret
Then the system must:
  | Step                        | Validation                                  |
  | Verify authorisation       | Check role and resource permissions         |
  | Log access attempt         | Before retrieval, include purpose           |
  | Retrieve encrypted data    | From secure storage                         |
  | Verify integrity           | Check authentication tag                    |
  | Decrypt DEK               | Using current KEK                           |
  | Decrypt secret            | Using DEK and verify                        |
  | Clear memory              | Zero all intermediate values                |
And handle failures by:
  - Logging detailed error internally
  - Returning generic error to caller
  - Alerting on repeated failures
  - Locking after threshold exceeded
```

### Scenario: Key Rotation Workflow
```gherkin
Given keys must be rotated periodically
When rotation is triggered
Then execute the following workflow:
  | Phase              | Actions                                          |
  | Preparation       | Generate new KEK, keep old active               |
  | Re-encryption     | Decrypt with old KEK, encrypt with new          |
  | Verification      | Test decryption with new KEK                    |
  | Activation        | Mark new KEK as primary                         |
  | Cleanup           | Remove old KEK after grace period               |
And ensure:
  - Zero downtime during rotation
  - Ability to rollback if issues
  - Progress tracking and resumption
  - Audit trail of all operations
```

### Scenario: Hardware Security Module (HSM) Integration
```gherkin
Given HSM provides hardware-based key protection
When HSM is available
Then integrate using:
  | Operation                  | HSM Usage                                    |
  | Master key storage        | Store KEK in HSM, never extract             |
  | Key generation            | Generate keys within HSM                     |
  | Signing operations        | Perform in HSM, return signature only       |
  | Key wrapping              | Wrap DEKs using HSM master key              |
  | Load balancing            | Distribute across multiple HSMs              |
And support fallback to software when:
  - HSM is initialising
  - HSM is overloaded (with backpressure)
  - Development/test environments
```

### Scenario: Secret Access Patterns
```gherkin
Given secrets have different access patterns
When implementing caching
Then apply appropriate strategies:
  | Secret Type          | Cache Strategy     | TTL    | Invalidation           |
  | Frequently used     | Memory cache       | 5 min  | On rotation/revoke     |
  | Sensitive keys      | No caching         | N/A    | Always fetch          |
  | System configs      | Encrypted cache    | 1 hour | On update             |
  | Session keys        | Distributed cache  | 15 min | On logout             |
And implement cache security:
  - Encrypt cached values in memory
  - Clear on application shutdown
  - Monitor for cache timing attacks
```

### Scenario: Emergency Access Procedures
```gherkin
Given emergencies may require special access
When break-glass procedure is invoked
Then the system must:
  | Step                      | Action                                       |
  | Authenticate             | Require 2-factor authentication              |
  | Verify authority         | Check emergency access list                  |
  | Log prominently          | Create high-priority audit entry             |
  | Grant temporary access   | Time-limited elevated privileges             |
  | Notify stakeholders      | Alert security team immediately              |
  | Monitor actions          | Log every operation performed                |
  | Revoke automatically     | After time limit or manual revocation        |
And require post-incident review
```

### Scenario: Secret Sharing and Escrow
```gherkin
Given some secrets need recovery options
When implementing key escrow
Then use Shamir's Secret Sharing:
  | Configuration           | Value                                         |
  | Total shares           | 5                                             |
  | Threshold              | 3 shares required                             |
  | Share distribution     | Different geographic locations                |
  | Share protection       | Each share encrypted separately               |
  | Verification           | Can reconstruct without exposing              |
And implement secure distribution:
  - Generate shares in secure environment
  - Transmit via separate channels
  - Store with different custodians
  - Test recovery annually
```

### Scenario: Compliance and Audit Requirements
```gherkin
Given regulatory requirements for key management
When implementing audit trails
Then capture:
  | Event                    | Details Logged                               |
  | Key generation          | Algorithm, size, purpose, creator            |
  | Key usage               | Operation, timestamp, user, system           |
  | Key rotation            | Old ID, new ID, reason, performer           |
  | Access attempt          | Success/failure, user, IP, purpose          |
  | Policy violation        | Rule violated, action taken, context         |
And provide compliance reports:
  - Key inventory with algorithms
  - Rotation compliance status
  - Access frequency analysis
  - Policy violation summary
```

## Edge Cases and Security Considerations

### Edge Case: KEK Corruption or Loss
```gherkin
Given the KEK could be corrupted
When corruption is detected
Then:
  - Alert immediately to security team
  - Attempt recovery from backups
  - If unrecoverable, initiate disaster recovery
  - Document lessons learned
  - Require all certificates to be reissued
```

### Edge Case: Memory Disclosure Attacks
```gherkin
Given secrets exist in memory temporarily
When handling sensitive data
Then:
  - Use memory-safe operations
  - Clear memory immediately after use
  - Disable core dumps for the process
  - Use mlock() to prevent swapping
  - Implement memory guards
```

### Edge Case: Concurrent Access Under Load
```gherkin
Given high load may cause contention
When multiple requests need the same secret
Then:
  - Implement read-write locks appropriately
  - Queue requests to prevent thundering herd
  - Monitor for timing attacks
  - Ensure consistent performance
```

## Security Hardening Measures

```gherkin
Given defence-in-depth is required
Then implement these additional measures:
  | Measure                      | Implementation                              |
  | Key stretching              | PBKDF2 with 100,000+ iterations           |
  | Salt generation             | Cryptographically random, unique per key    |
  | Timing attack prevention    | Constant-time comparisons                   |
  | Side-channel protection     | Add random delays for sensitive ops         |
  | Quantum resistance prep     | Plan for post-quantum algorithm migration   |
```

## Technical Notes
- Use established cryptographic libraries only (cryptography.io)
- Implement FIPS 140-2 compliant modes where required
- Consider using Vault or similar for complex deployments
- Plan for secure deletion on various storage media
- Implement rate limiting on decryption operations
- Monitor for unusual access patterns
- Use secure random number generation (/dev/urandom or better)

## Definition of Done
- [ ] KEK initialisation and protection implemented
- [ ] Private key encryption with unique DEKs
- [ ] Secret categorisation and policies enforced
- [ ] Secure retrieval with audit logging
- [ ] Key rotation workflow automated
- [ ] HSM integration prepared (interfaces defined)
- [ ] Caching strategies implemented securely
- [ ] Emergency access procedures documented
- [ ] Secret sharing for recovery implemented
- [ ] Compliance reporting available
- [ ] All edge cases handled
- [ ] Security scan passing
- [ ] Cryptographic operations benchmarked
- [ ] Disaster recovery plan tested
- [ ] Zero sensitive data in logs verified