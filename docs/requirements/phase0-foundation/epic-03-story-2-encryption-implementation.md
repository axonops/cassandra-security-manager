# User Story 03.2: Encryption Implementation

## Story
As a **security engineer**, I want to **implement comprehensive encryption for Cassandra** so that **all certificate data is protected at rest and in transit**.

## Acceptance Criteria
```gherkin
Feature: Cassandra Encryption Implementation
  As a security engineer
  I want to encrypt all data at rest and in transit
  So that certificate data remains confidential

  Scenario: Inter-node encryption setup
    Given I have a multi-node Cassandra cluster
    When I enable inter-node encryption with TLS 1.3
    Then all node-to-node communication should be encrypted
    And older TLS versions should be rejected
    And certificate validation should be enforced

  Scenario: Client-to-node encryption
    Given I have client applications connecting to Cassandra
    When I enable client encryption with mutual TLS
    Then all client connections should use TLS 1.3
    And clients without valid certificates should be rejected
    And connection logs should show encryption details

  Scenario: Encryption at rest
    Given I have sensitive certificate data in Cassandra
    When I enable transparent data encryption
    Then all SSTables should be encrypted on disk
    And commit logs should be encrypted
    And encryption should not impact read performance by more than 5%

  Scenario: Key rotation
    Given I have encrypted data with current keys
    When I initiate a key rotation procedure
    Then new data should use the new keys
    And old data should remain readable
    And the rotation should complete without downtime
```

## Technical Details

### SSL/TLS Configuration
```yaml
# cassandra.yaml encryption settings
server_encryption_options:
  internode_encryption: all
  keystore: /etc/cassandra/ssl/cassandra.keystore
  keystore_password: ${KEYSTORE_PASSWORD}
  truststore: /etc/cassandra/ssl/cassandra.truststore
  truststore_password: ${TRUSTSTORE_PASSWORD}
  protocol: TLSv1.3
  cipher_suites:
    - TLS_AES_256_GCM_SHA384
    - TLS_AES_128_GCM_SHA256
    - TLS_CHACHA20_POLY1305_SHA256
  require_client_auth: true
  require_endpoint_verification: true

client_encryption_options:
  enabled: true
  keystore: /etc/cassandra/ssl/cassandra.keystore
  keystore_password: ${KEYSTORE_PASSWORD}
  truststore: /etc/cassandra/ssl/cassandra.truststore
  truststore_password: ${TRUSTSTORE_PASSWORD}
  protocol: TLSv1.3
  cipher_suites:
    - TLS_AES_256_GCM_SHA384
    - TLS_AES_128_GCM_SHA256
  require_client_auth: true
```

### Transparent Data Encryption (TDE)
```yaml
# Enable encryption at rest
transparent_data_encryption_options:
  enabled: true
  cipher: AES/CBC/PKCS5Padding
  chunk_length_kb: 64
  key_alias: ${ENCRYPTION_KEY_ALIAS}
  key_provider:
    - class_name: org.apache.cassandra.security.JKSKeyProvider
      parameters:
        - keystore: /etc/cassandra/ssl/encryption.keystore
        - keystore_password: ${ENCRYPTION_KEYSTORE_PASSWORD}
        - store_type: JCEKS
        - key_password: ${ENCRYPTION_KEY_PASSWORD}
```

### Key Management Implementation
```python
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.backends import default_backend
from cryptography.x509 import CertificateBuilder, Name, NameAttribute
from cryptography.x509.oid import NameOID
from datetime import datetime, timedelta
import os
import hashlib

class CassandraEncryptionManager:
    def __init__(self, config):
        self.config = config
        self.backend = default_backend()
        
    def generate_node_certificate(self, node_name: str):
        """Generate TLS certificate for Cassandra node"""
        # Generate private key
        private_key = rsa.generate_private_key(
            public_exponent=65537,
            key_size=4096,
            backend=self.backend
        )
        
        # Build certificate
        subject = Name([
            NameAttribute(NameOID.COMMON_NAME, node_name),
            NameAttribute(NameOID.ORGANIZATION_NAME, "Cassandra Security Manager"),
            NameAttribute(NameOID.ORGANIZATIONAL_UNIT_NAME, "Database Cluster")
        ])
        
        builder = CertificateBuilder()
        builder = builder.subject_name(subject)
        builder = builder.issuer_name(subject)  # Self-signed for now
        builder = builder.public_key(private_key.public_key())
        builder = builder.serial_number(int.from_bytes(os.urandom(16), 'big'))
        builder = builder.not_valid_before(datetime.utcnow())
        builder = builder.not_valid_after(datetime.utcnow() + timedelta(days=365))
        
        # Add SAN for node addresses
        builder = builder.add_extension(
            x509.SubjectAlternativeName([
                x509.DNSName(node_name),
                x509.DNSName(f"{node_name}.cassandra.local"),
                x509.IPAddress(ipaddress.ip_address("127.0.0.1"))
            ]),
            critical=False
        )
        
        certificate = builder.sign(private_key, hashes.SHA384(), self.backend)
        
        return private_key, certificate
    
    def create_keystore(self, node_name: str, output_path: str):
        """Create Java keystore for Cassandra node"""
        private_key, certificate = self.generate_node_certificate(node_name)
        
        # Convert to PKCS12 format
        pkcs12 = serialization.pkcs12.serialize_key_and_certificates(
            name=node_name.encode(),
            key=private_key,
            cert=certificate,
            cas=None,
            encryption_algorithm=serialization.BestAvailableEncryption(
                self.config['keystore_password'].encode()
            )
        )
        
        # Write keystore
        keystore_path = os.path.join(output_path, f"{node_name}.p12")
        with open(keystore_path, 'wb') as f:
            f.write(pkcs12)
            
        # Convert to JKS using keytool
        self._convert_to_jks(keystore_path, node_name)
        
    def generate_encryption_key(self):
        """Generate data encryption key for TDE"""
        key = os.urandom(32)  # 256-bit key
        return base64.b64encode(key).decode('utf-8')
    
    def rotate_encryption_keys(self):
        """Implement key rotation procedure"""
        # Generate new key
        new_key = self.generate_encryption_key()
        
        # Update key alias in configuration
        new_alias = f"encryption_key_{datetime.utcnow().strftime('%Y%m%d%H%M%S')}"
        
        # Add new key to keystore
        self._add_key_to_keystore(new_key, new_alias)
        
        # Update Cassandra configuration
        self._update_cassandra_config(new_alias)
        
        # Trigger online key rotation
        self._trigger_key_rotation()
```

### Encryption Monitoring
```python
class EncryptionMonitor:
    def __init__(self, cassandra_client):
        self.client = cassandra_client
        
    def verify_encryption_status(self):
        """Verify all encryption is active"""
        checks = {
            'inter_node_encryption': self._check_inter_node_encryption(),
            'client_encryption': self._check_client_encryption(),
            'data_at_rest_encryption': self._check_data_encryption(),
            'key_rotation_status': self._check_key_rotation()
        }
        
        return all(checks.values()), checks
    
    def _check_inter_node_encryption(self):
        """Verify inter-node encryption is active"""
        result = self.client.execute(
            "SELECT * FROM system_views.settings WHERE name = 'server_encryption_options'"
        )
        settings = result.one()
        return settings and 'internode_encryption: all' in settings.value
    
    def _check_client_encryption(self):
        """Verify client encryption is enabled"""
        result = self.client.execute(
            "SELECT * FROM system_views.clients"
        )
        for client in result:
            if not client.ssl_cipher_suite:
                return False
        return True
    
    def get_encryption_metrics(self):
        """Collect encryption performance metrics"""
        return {
            'encrypted_connections': self._count_encrypted_connections(),
            'encryption_handshake_time_ms': self._measure_handshake_time(),
            'key_rotation_last_run': self._get_last_key_rotation(),
            'encrypted_sstables_percentage': self._calculate_encrypted_percentage()
        }
```

## Security Requirements

### Certificate Management
1. **Certificate Generation**
   - 4096-bit RSA keys minimum
   - SHA-384 signature algorithm
   - 1-year validity for node certificates
   - Unique certificates per node

2. **Certificate Validation**
   - Hostname verification enabled
   - Certificate chain validation
   - Revocation checking via OCSP
   - Certificate pinning for critical connections

### Key Management
1. **Key Storage**
   - Hardware Security Module (HSM) integration ready
   - Encrypted key storage
   - Secure key backup procedures
   - Key escrow capabilities

2. **Key Rotation**
   - Automated rotation schedules
   - Zero-downtime rotation
   - Audit trail for all key operations
   - Rollback capabilities

## Testing Requirements
1. **Security Tests**
   - TLS configuration scanning
   - Cipher suite validation
   - Certificate verification tests
   - Encryption strength validation

2. **Performance Tests**
   - Encryption overhead measurement
   - Key rotation performance
   - Connection establishment timing
   - Throughput with encryption enabled

## Definition of Done
- [ ] Inter-node encryption configured and verified
- [ ] Client encryption with mutual TLS working
- [ ] Transparent data encryption enabled
- [ ] Key rotation procedure tested
- [ ] Performance impact documented
- [ ] Security scan shows no vulnerabilities
- [ ] Monitoring dashboards show encryption status
- [ ] Documentation includes security guide