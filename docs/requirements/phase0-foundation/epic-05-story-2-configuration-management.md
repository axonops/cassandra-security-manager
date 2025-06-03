# 05.2 - Configuration Management

## User Story
**As a** system administrator  
**I want** flexible configuration options  
**So that** I can adapt the system to my environment without code changes

## INVEST Criteria
- **Independent**: Configuration system can be built independently
- **Negotiable**: Specific configuration formats can be adjusted
- **Valuable**: Essential for deployments across environments
- **Estimable**: 1 sprint for comprehensive implementation
- **Small**: Focused on configuration infrastructure
- **Testable**: Configuration validation can be automated

## Acceptance Criteria

### Scenario: Configuration Source Hierarchy
```gherkin
Given multiple configuration sources exist
When the application loads configuration
Then apply the following precedence (highest to lowest):
  | Priority | Source                    | Example                            | Use Case           |
  | 1        | Command line arguments   | --api-port=8443                   | Quick overrides    |
  | 2        | Environment variables    | CSM_API_PORT=8443                 | Container deploys  |
  | 3        | External config service  | Consul/Vault/K8s ConfigMap        | Dynamic updates    |
  | 4        | Local config file        | /etc/csm/config.yaml              | Server installs    |
  | 5        | User config file         | ~/.csm/config.yaml                | Development        |
  | 6        | Application defaults     | Internal defaults                  | Fallback values    |
And merge configurations deeply:
  - Override leaf values only
  - Preserve unspecified values
  - Log configuration sources
  - Validate after merging
```

### Scenario: Configuration File Formats
```gherkin
Given different teams prefer different formats
When loading configuration files
Then support multiple formats:
  | Format | Extension | Example                                          |
  | YAML   | .yaml     | database:\n  host: localhost\n  port: 9042     |
  | JSON   | .json     | {"database": {"host": "localhost"}}            |
  | TOML   | .toml     | [database]\nhost = "localhost"                 |
  | ENV    | .env      | DATABASE_HOST=localhost                         |
And handle format-specific features:
  - YAML anchors and references
  - JSON schema validation
  - TOML tables and arrays
  - ENV file variable expansion
```

### Scenario: Environment Variable Mapping
```gherkin
Given environment variables are preferred in containers
When mapping environment variables to config
Then follow consistent naming:
  | Config Path                 | Environment Variable        | Value Type |
  | api.port                   | CSM_API_PORT               | integer    |
  | api.tls.enabled            | CSM_API_TLS_ENABLED        | boolean    |
  | database.hosts[0]          | CSM_DATABASE_HOSTS_0       | string     |
  | database.connection_pool   | CSM_DATABASE_CONNECTION_POOL| json      |
  | security.jwt.secret        | CSM_SECURITY_JWT_SECRET    | string     |
And support complex types:
  - JSON for objects: CSM_LABELS='{"env":"prod","team":"security"}'
  - Comma-separated lists: CSM_DATABASE_HOSTS="host1,host2,host3"
  - Base64 for binary: CSM_TLS_CERT="base64:LS0tLS1..."
```

### Scenario: Configuration Validation
```gherkin
Given invalid configuration causes failures
When configuration is loaded
Then validate all settings:
  | Validation Type        | Example                                    | Action on Failure |
  | Required fields       | api.port must exist                        | Exit with error   |
  | Type checking         | port must be integer 1-65535              | Exit with error   |
  | Format validation     | email must match RFC 5322                  | Exit with error   |
  | Range validation      | timeout must be 1-300 seconds              | Exit with error   |
  | Mutual exclusivity    | Cannot set both HSM and file key storage   | Exit with error   |
  | Dependency checking   | If TLS enabled, cert path required         | Exit with error   |
  | Security validation   | Passwords meet complexity requirements      | Exit with error   |
And provide helpful error messages:
"""
Configuration error at 'database.connection_pool.max_size':
  Value: 1000
  Error: Must be between 1 and 100
  Help: Consider using multiple connection pools for high load
"""
```

### Scenario: Secure Configuration Handling
```gherkin
Given configuration contains secrets
When handling sensitive values
Then implement security measures:
  | Secret Type          | Storage Method              | Access Method               |
  | Passwords           | Environment variable         | One-time read, then clear   |
  | API keys            | External secret manager      | Fetch on demand             |
  | Certificates        | File path reference          | Load and validate           |
  | Private keys        | HSM reference               | Never load into memory      |
  | Encryption keys     | Key derivation              | Derive from master          |
And prevent exposure:
  - Redact secrets in logs: "password: ***REDACTED***"
  - Exclude from config dumps
  - Clear from environment after reading
  - Audit access to secrets
```

### Scenario: Dynamic Configuration Updates
```gherkin
Given some settings should update without restart
When configuration changes are detected
Then handle updates appropriately:
  | Setting Category     | Update Method        | Example Settings              |
  | Feature flags       | Hot reload           | enable_new_algorithm          |
  | Rate limits         | Hot reload           | api_rate_limit_per_minute     |
  | Log levels          | Hot reload           | log_level, component_levels   |
  | Cache settings      | Hot reload           | cache_ttl, cache_size         |
  | Database pools      | Graceful recreate    | pool_size, timeout            |
  | TLS certificates    | Graceful reload      | cert_path, key_path           |
  | API endpoints       | Requires restart     | api_port, bind_address        |
  | Security settings   | Requires restart     | jwt_algorithm, auth_providers |
And implement safe updates:
  - Validate new config before applying
  - Atomic updates only
  - Rollback on failure
  - Log all changes
  - Notify dependent services
```

### Scenario: Multi-Environment Configuration
```gherkin
Given different environments have different needs
When organising configuration
Then support environment-specific settings:
  | Environment | Config File              | Purpose                    |
  | default     | config.default.yaml      | Base configuration         |
  | development | config.development.yaml  | Local development          |
  | test        | config.test.yaml         | Automated testing          |
  | staging     | config.staging.yaml      | Pre-production            |
  | production  | config.production.yaml   | Production settings        |
And enforce environment rules:
  - Development: Allow insecure options
  - Test: Fast timeouts, in-memory storage
  - Staging: Production-like with debugging
  - Production: Strict security, no debugging
```

### Scenario: Configuration Documentation
```gherkin
Given operators need configuration reference
When generating documentation
Then provide comprehensive information:
  | Section              | Content                                      |
  | Overview            | Purpose and configuration sources            |
  | Complete reference  | All settings with types and defaults         |
  | Examples            | Common configuration scenarios               |
  | Environment mapping | Config key to env var mapping                |
  | Migration guide     | Changes between versions                     |
And auto-generate from code:
  - Extract from config schemas
  - Include validation rules
  - Show relationships
  - Generate in multiple formats (MD, HTML, man)
```

### Scenario: Configuration Templates
```gherkin
Given common deployment patterns exist
When providing configuration examples
Then include templates for:
  | Deployment Type     | Template Features                            |
  | Single node         | Minimal config, file storage                 |
  | High availability   | Multi-node, load balancing                   |
  | Air-gapped          | No external dependencies                     |
  | Cloud native        | Service discovery, object storage            |
  | Enterprise          | LDAP, HSM, external secrets                  |
And include tooling:
  - Config generator wizard
  - Migration scripts
  - Validation tools
  - Diff utilities
```

### Scenario: Configuration Audit Trail
```gherkin
Given configuration changes affect security
When configuration is modified
Then maintain audit trail:
  | Event                  | Logged Information                          |
  | Initial load          | Source files, resolved values                |
  | Runtime update        | Changed values, who, when, why               |
  | Validation failure    | Invalid values, reason                       |
  | Secret access         | Which secrets, by whom                       |
  | Environment override  | Original vs override values                  |
And provide configuration history:
  - Store config snapshots
  - Track changes over time
  - Compare configurations
  - Rollback capabilities
```

## Edge Cases and Configuration Considerations

### Edge Case: Missing Required Configuration
```gherkin
Given required configuration is missing
When application starts
Then:
  - Check for common mistakes (typos, wrong path)
  - Suggest likely fixes
  - Provide example configuration
  - Exit with clear error message
  - Return non-zero exit code
```

### Edge Case: Configuration Service Unavailable
```gherkin
Given external config service is used
When service is unavailable at startup
Then:
  - Retry with exponential backoff
  - Fall back to cached configuration
  - Start in degraded mode if possible
  - Alert operators
  - Document minimum required config
```

### Edge Case: Circular Configuration Dependencies
```gherkin
Given configuration can reference other values
When circular reference is detected
Then:
  - Detect cycle during validation
  - Report all values in cycle
  - Suggest resolution
  - Fail fast with clear error
```

### Edge Case: Large Configuration Files
```gherkin
Given configuration might be extensive
When loading large configs
Then:
  - Stream parse when possible
  - Set reasonable size limits
  - Validate incrementally
  - Provide progress feedback
  - Support configuration includes
```

## Security Hardening

```gherkin
Given configuration is attack surface
Then implement protections:
  | Threat                    | Mitigation                                |
  | Path traversal           | Validate all file paths                    |
  | Template injection       | Limit variable expansion                   |
  | XXE attacks              | Disable external entities                  |
  | Secret exposure          | Audit all config access                    |
  | Privilege escalation     | Validate permission changes                |
```

## Technical Notes
- Use Pydantic for configuration models
- Implement JSON Schema for validation
- Support config hot-reload via file watching
- Use python-dotenv for .env files
- Implement config diffing for updates
- Cache parsed configuration
- Support config imports/includes
- Implement dry-run mode for validation

## Definition of Done
- [ ] Configuration source hierarchy implemented
- [ ] Multiple file formats supported (YAML, JSON, TOML, ENV)
- [ ] Environment variable mapping with prefixes
- [ ] Comprehensive validation with helpful errors
- [ ] Secure secret handling with redaction
- [ ] Dynamic updates for applicable settings
- [ ] Environment-specific configurations
- [ ] Auto-generated documentation
- [ ] Configuration templates provided
- [ ] Audit trail for all changes
- [ ] Edge cases handled gracefully
- [ ] Security hardening applied
- [ ] Unit tests for all scenarios
- [ ] Integration tests with real configs
- [ ] Performance tests for large configs
- [ ] Configuration migration tools
- [ ] Operator documentation complete