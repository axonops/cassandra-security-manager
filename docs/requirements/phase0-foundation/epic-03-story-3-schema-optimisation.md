# User Story 03.3: Schema Optimisation

## Story
As a **database architect**, I want to **design optimised Cassandra schemas** so that **certificate operations are fast and scalable**.

## Acceptance Criteria
```gherkin
Feature: Optimised Schema Design
  As a database architect
  I want to create efficient Cassandra schemas
  So that certificate operations perform at scale

  Scenario: Certificate storage efficiency
    Given I need to store millions of certificates
    When I query certificates by various criteria
    Then queries should return in under 10ms
    And there should be no partition hot spots
    And storage should be space-efficient

  Scenario: Audit log performance
    Given I need to write thousands of audit events per second
    When I write audit logs continuously
    Then writes should not create bottlenecks
    And older logs should be automatically expired
    And queries by time range should be efficient

  Scenario: Search capabilities
    Given I need to search certificates by multiple attributes
    When I search by subject, issuer, or expiry date
    Then results should return within 100ms
    And pagination should work consistently
    And no full table scans should occur

  Scenario: Schema evolution
    Given I need to add new fields to existing tables
    When I apply schema migrations
    Then migrations should complete without downtime
    And existing data should remain accessible
    And new fields should have sensible defaults
```

## Technical Details

### Keyspace Design
```sql
-- Main keyspace with appropriate replication
CREATE KEYSPACE IF NOT EXISTS certificate_manager
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'dc1': 3
}
AND durable_writes = true;

-- Separate keyspace for audit logs with different retention
CREATE KEYSPACE IF NOT EXISTS audit_logs
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'dc1': 2
}
AND durable_writes = true;
```

### Core Schema Design
```sql
-- Certificates table optimised for common queries
CREATE TABLE IF NOT EXISTS certificate_manager.certificates (
    organisation_id UUID,
    certificate_id UUID,
    serial_number TEXT,
    common_name TEXT,
    subject_dn TEXT,
    issuer_dn TEXT,
    not_before TIMESTAMP,
    not_after TIMESTAMP,
    certificate_type TEXT,
    key_algorithm TEXT,
    key_size INT,
    signature_algorithm TEXT,
    certificate_pem TEXT,
    certificate_der BLOB,
    private_key_encrypted BLOB,
    key_encryption_key_id UUID,
    status TEXT,
    created_at TIMESTAMP,
    created_by UUID,
    updated_at TIMESTAMP,
    metadata MAP<TEXT, TEXT>,
    tags SET<TEXT>,
    PRIMARY KEY ((organisation_id), not_after, certificate_id)
) WITH CLUSTERING ORDER BY (not_after ASC, certificate_id DESC)
AND default_time_to_live = 94608000  -- 3 years
AND gc_grace_seconds = 864000
AND compaction = {
    'class': 'TimeWindowCompactionStrategy',
    'compaction_window_unit': 'DAYS',
    'compaction_window_size': 1
};

-- Index for certificate lookups by serial number
CREATE MATERIALIZED VIEW IF NOT EXISTS certificate_manager.certificates_by_serial AS
    SELECT * FROM certificate_manager.certificates
    WHERE organisation_id IS NOT NULL
    AND serial_number IS NOT NULL
    AND not_after IS NOT NULL
    AND certificate_id IS NOT NULL
    PRIMARY KEY ((organisation_id, serial_number), not_after, certificate_id)
    WITH CLUSTERING ORDER BY (not_after ASC, certificate_id DESC);

-- Index for certificate lookups by common name
CREATE MATERIALIZED VIEW IF NOT EXISTS certificate_manager.certificates_by_cn AS
    SELECT * FROM certificate_manager.certificates
    WHERE organisation_id IS NOT NULL
    AND common_name IS NOT NULL
    AND not_after IS NOT NULL
    AND certificate_id IS NOT NULL
    PRIMARY KEY ((organisation_id, common_name), not_after, certificate_id)
    WITH CLUSTERING ORDER BY (not_after ASC, certificate_id DESC);

-- Certificate chains for hierarchy
CREATE TABLE IF NOT EXISTS certificate_manager.certificate_chains (
    organisation_id UUID,
    certificate_id UUID,
    chain_order INT,
    parent_certificate_id UUID,
    certificate_pem TEXT,
    PRIMARY KEY ((organisation_id, certificate_id), chain_order)
) WITH CLUSTERING ORDER BY (chain_order ASC);

-- Audit log table with time-based partitioning
CREATE TABLE IF NOT EXISTS audit_logs.events (
    date_bucket DATE,
    event_time TIMESTAMP,
    event_id UUID,
    organisation_id UUID,
    user_id UUID,
    event_type TEXT,
    resource_type TEXT,
    resource_id UUID,
    action TEXT,
    result TEXT,
    ip_address INET,
    user_agent TEXT,
    details TEXT,
    metadata MAP<TEXT, TEXT>,
    PRIMARY KEY ((date_bucket, organisation_id), event_time, event_id)
) WITH CLUSTERING ORDER BY (event_time DESC, event_id DESC)
AND default_time_to_live = 7776000  -- 90 days
AND compaction = {
    'class': 'TimeWindowCompactionStrategy',
    'compaction_window_unit': 'HOURS',
    'compaction_window_size': 1
};

-- Certificate metadata for flexible querying
CREATE TABLE IF NOT EXISTS certificate_manager.certificate_metadata (
    organisation_id UUID,
    metadata_type TEXT,
    metadata_value TEXT,
    certificate_id UUID,
    not_after TIMESTAMP,
    PRIMARY KEY ((organisation_id, metadata_type, metadata_value), not_after, certificate_id)
) WITH CLUSTERING ORDER BY (not_after ASC, certificate_id DESC);

-- Secure connection bundles
CREATE TABLE IF NOT EXISTS certificate_manager.connection_bundles (
    organisation_id UUID,
    bundle_id UUID,
    bundle_name TEXT,
    cluster_name TEXT,
    datacenter TEXT,
    contact_points SET<TEXT>,
    ca_certificate_id UUID,
    client_certificate_id UUID,
    created_at TIMESTAMP,
    expires_at TIMESTAMP,
    bundle_data BLOB,
    checksum TEXT,
    status TEXT,
    PRIMARY KEY ((organisation_id), bundle_name, bundle_id)
) WITH CLUSTERING ORDER BY (bundle_name ASC, bundle_id DESC);
```

### Query Patterns Implementation
```python
from cassandra.cluster import Cluster
from cassandra.policies import DCAwareRoundRobinPolicy
from cassandra.query import SimpleStatement, ConsistencyLevel
import uuid
from datetime import datetime, timedelta

class CertificateRepository:
    def __init__(self, contact_points, datacenter='dc1'):
        self.cluster = Cluster(
            contact_points,
            load_balancing_policy=DCAwareRoundRobinPolicy(local_dc=datacenter)
        )
        self.session = self.cluster.connect()
        self._prepare_statements()
    
    def _prepare_statements(self):
        """Prepare all statements for optimal performance"""
        self.insert_cert_stmt = self.session.prepare("""
            INSERT INTO certificate_manager.certificates (
                organisation_id, certificate_id, serial_number,
                common_name, subject_dn, issuer_dn,
                not_before, not_after, certificate_type,
                key_algorithm, key_size, signature_algorithm,
                certificate_pem, certificate_der, private_key_encrypted,
                key_encryption_key_id, status, created_at,
                created_by, updated_at, metadata, tags
            ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        """)
        
        self.get_expiring_certs_stmt = self.session.prepare("""
            SELECT * FROM certificate_manager.certificates
            WHERE organisation_id = ?
            AND not_after >= ?
            AND not_after <= ?
            LIMIT ?
        """)
        
        self.search_by_metadata_stmt = self.session.prepare("""
            SELECT certificate_id, not_after
            FROM certificate_manager.certificate_metadata
            WHERE organisation_id = ?
            AND metadata_type = ?
            AND metadata_value = ?
            LIMIT ?
        """)
    
    def save_certificate(self, cert_data):
        """Save certificate with optimal batching"""
        # Main certificate insert
        self.session.execute(self.insert_cert_stmt, [
            cert_data['organisation_id'],
            cert_data['certificate_id'],
            cert_data['serial_number'],
            cert_data['common_name'],
            cert_data['subject_dn'],
            cert_data['issuer_dn'],
            cert_data['not_before'],
            cert_data['not_after'],
            cert_data['certificate_type'],
            cert_data['key_algorithm'],
            cert_data['key_size'],
            cert_data['signature_algorithm'],
            cert_data['certificate_pem'],
            cert_data['certificate_der'],
            cert_data['private_key_encrypted'],
            cert_data['key_encryption_key_id'],
            'ACTIVE',
            datetime.utcnow(),
            cert_data['created_by'],
            datetime.utcnow(),
            cert_data.get('metadata', {}),
            cert_data.get('tags', set())
        ])
        
        # Index metadata for searching
        if cert_data.get('metadata'):
            self._index_metadata(cert_data)
    
    def find_expiring_certificates(self, organisation_id, days_ahead=30, limit=1000):
        """Find certificates expiring within specified days"""
        now = datetime.utcnow()
        expiry_window = now + timedelta(days=days_ahead)
        
        return self.session.execute(
            self.get_expiring_certs_stmt,
            [organisation_id, now, expiry_window, limit]
        )
    
    def search_by_metadata(self, organisation_id, metadata_type, metadata_value, limit=100):
        """Search certificates by metadata attributes"""
        cert_refs = self.session.execute(
            self.search_by_metadata_stmt,
            [organisation_id, metadata_type, metadata_value, limit]
        )
        
        # Fetch full certificate details
        certificates = []
        for ref in cert_refs:
            cert = self._get_certificate_by_id(
                organisation_id,
                ref.certificate_id,
                ref.not_after
            )
            if cert:
                certificates.append(cert)
        
        return certificates
```

### Performance Optimisation
```python
class SchemaOptimiser:
    def __init__(self, session):
        self.session = session
    
    def analyse_partition_sizes(self, keyspace, table):
        """Analyse partition sizes to detect hot spots"""
        query = """
            SELECT partition_key, 
                   mean_partition_size, 
                   max_partition_size,
                   partition_count
            FROM system.size_estimates
            WHERE keyspace_name = %s AND table_name = %s
        """
        
        results = self.session.execute(query, [keyspace, table])
        
        hot_partitions = []
        for row in results:
            if row.max_partition_size > 100 * 1024 * 1024:  # 100MB
                hot_partitions.append({
                    'partition_key': row.partition_key,
                    'size': row.max_partition_size,
                    'count': row.partition_count
                })
        
        return hot_partitions
    
    def optimise_compaction(self, keyspace, table, strategy='TWCS'):
        """Optimise table compaction strategy"""
        if strategy == 'TWCS':
            alter_query = f"""
                ALTER TABLE {keyspace}.{table}
                WITH compaction = {{
                    'class': 'TimeWindowCompactionStrategy',
                    'compaction_window_unit': 'DAYS',
                    'compaction_window_size': 1,
                    'min_threshold': 4,
                    'max_threshold': 32
                }}
            """
        elif strategy == 'LCS':
            alter_query = f"""
                ALTER TABLE {keyspace}.{table}
                WITH compaction = {{
                    'class': 'LeveledCompactionStrategy',
                    'sstable_size_in_mb': 160
                }}
            """
        
        self.session.execute(alter_query)
    
    def create_custom_index(self, field_name, cardinality='low'):
        """Create optimised secondary indices"""
        if cardinality == 'low':
            # Use SASI for low cardinality
            index_query = f"""
                CREATE CUSTOM INDEX ON certificate_manager.certificates ({field_name})
                USING 'org.apache.cassandra.index.sasi.SASIIndex'
                WITH OPTIONS = {{
                    'mode': 'PREFIX',
                    'case_sensitive': 'false'
                }}
            """
        else:
            # Use materialised view for high cardinality
            view_query = f"""
                CREATE MATERIALIZED VIEW certificate_manager.certificates_by_{field_name} AS
                SELECT * FROM certificate_manager.certificates
                WHERE {field_name} IS NOT NULL
                AND organisation_id IS NOT NULL
                AND not_after IS NOT NULL
                AND certificate_id IS NOT NULL
                PRIMARY KEY ((organisation_id, {field_name}), not_after, certificate_id)
            """
            self.session.execute(view_query)
```

## Migration Strategy

### Schema Versioning
```sql
-- Schema version tracking
CREATE TABLE IF NOT EXISTS certificate_manager.schema_versions (
    version INT,
    description TEXT,
    applied_at TIMESTAMP,
    applied_by TEXT,
    checksum TEXT,
    PRIMARY KEY (version)
);
```

### Migration Implementation
```python
class SchemaMigration:
    def __init__(self, session):
        self.session = session
        
    def apply_migration(self, version, migration_script):
        """Apply schema migration with safety checks"""
        # Check if already applied
        existing = self.session.execute(
            "SELECT * FROM certificate_manager.schema_versions WHERE version = %s",
            [version]
        ).one()
        
        if existing:
            raise Exception(f"Migration {version} already applied")
        
        # Apply migration
        try:
            self.session.execute(migration_script)
            
            # Record migration
            self.session.execute("""
                INSERT INTO certificate_manager.schema_versions
                (version, description, applied_at, applied_by, checksum)
                VALUES (%s, %s, %s, %s, %s)
            """, [
                version,
                f"Migration {version}",
                datetime.utcnow(),
                'system',
                hashlib.sha256(migration_script.encode()).hexdigest()
            ])
        except Exception as e:
            raise Exception(f"Migration {version} failed: {str(e)}")
```

## Testing Requirements
1. **Performance Tests**
   - Query latency under load
   - Write throughput testing
   - Partition size monitoring
   - Compaction performance

2. **Schema Tests**
   - Migration rollback testing
   - Data integrity verification
   - Index performance validation
   - Consistency level testing

## Definition of Done
- [ ] All schemas created and documented
- [ ] Query patterns optimised and tested
- [ ] No partition hot spots detected
- [ ] Materialised views working correctly
- [ ] Migration framework implemented
- [ ] Performance benchmarks meet targets
- [ ] Monitoring queries implemented
- [ ] Schema documentation complete