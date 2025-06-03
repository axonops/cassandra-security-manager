# User Stor 03.1: Containerised Cassandra Deployment

## Story
As a **DevOps engineer**, I want to **deploy Cassandra in containers** so that **I can easily manage and scale the certificate management database**.

## Acceptance Criteria
```gherkin
Feature: Containerised Cassandra Deployment
  As a DevOps engineer
  I want to deploy Cassandra in Docker containers
  So that I can manage the database infrastructure efficiently

  Scenario: Single-node development deployment
    Given I have Docker installed on my development machine
    When I run the Cassandra container with development configuration
    Then Cassandra should start within 30 seconds
    And the health check endpoint should return healthy
    And I should be able to connect via cqlsh

  Scenario: Multi-node cluster formation
    Given I have three Cassandra container instances
    When I start them with cluster configuration
    Then they should discover each other automatically
    And form a three-node cluster
    And the nodetool status should show all nodes as UN (Up/Normal)

  Scenario: Persistent data storage
    Given I have a Cassandra container with mounted volumes
    When I write data to the database
    And restart the container
    Then the data should persist across restarts
    And no data loss should occur

  Scenario: Container health monitoring
    Given a running Cassandra container
    When I query the health check endpoint
    Then it should return the node status
    And indicate if the node is ready for traffic
    And report cluster health metrics
```

## Technical Details

### Dockerfile Implementation
```dockerfile
FROM cassandra:4.1-jammy

# Install additional tools
RUN apt-get update && apt-get install -y \
    curl \
    jq \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

# Copy custom configuration
COPY cassandra.yaml /etc/cassandra/
COPY cassandra-rackdc.properties /etc/cassandra/
COPY jvm.options /etc/cassandra/

# Install health check script
COPY health-check.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/health-check.sh

# Set up encryption keystores
RUN mkdir -p /etc/cassandra/ssl
COPY create-ssl-stores.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/create-ssl-stores.sh

# Expose ports
EXPOSE 7000 7001 7199 9042 9160

# Health check
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD /usr/local/bin/health-check.sh

# Volume for data persistence
VOLUME ["/var/lib/cassandra"]

# Entry point wrapper for initialisation
COPY docker-entrypoint-wrapper.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint-wrapper.sh
ENTRYPOINT ["/usr/local/bin/docker-entrypoint-wrapper.sh"]
```

### Docker Compose Configuration
```yaml
version: '3.8'

services:
  cassandra-seed:
    build: ./cassandra
    container_name: cassandra-seed
    hostname: cassandra-seed
    environment:
      - CASSANDRA_CLUSTER_NAME=CertificateManager
      - CASSANDRA_DC=dc1
      - CASSANDRA_RACK=rack1
      - CASSANDRA_SEEDS=cassandra-seed
      - CASSANDRA_ENDPOINT_SNITCH=GossipingPropertyFileSnitch
      - MAX_HEAP_SIZE=1G
      - HEAP_NEWSIZE=200M
    volumes:
      - cassandra-seed-data:/var/lib/cassandra
      - ./ssl/node1:/etc/cassandra/ssl
    ports:
      - "9042:9042"
    networks:
      - cassandra-net
    healthcheck:
      test: ["CMD", "/usr/local/bin/health-check.sh"]
      interval: 30s
      timeout: 10s
      retries: 5

  cassandra-node-1:
    build: ./cassandra
    container_name: cassandra-node-1
    hostname: cassandra-node-1
    depends_on:
      cassandra-seed:
        condition: service_healthy
    environment:
      - CASSANDRA_CLUSTER_NAME=CertificateManager
      - CASSANDRA_DC=dc1
      - CASSANDRA_RACK=rack2
      - CASSANDRA_SEEDS=cassandra-seed
      - CASSANDRA_ENDPOINT_SNITCH=GossipingPropertyFileSnitch
      - MAX_HEAP_SIZE=1G
      - HEAP_NEWSIZE=200M
    volumes:
      - cassandra-node-1-data:/var/lib/cassandra
      - ./ssl/node2:/etc/cassandra/ssl
    networks:
      - cassandra-net

  cassandra-node-2:
    build: ./cassandra
    container_name: cassandra-node-2
    hostname: cassandra-node-2
    depends_on:
      cassandra-seed:
        condition: service_healthy
    environment:
      - CASSANDRA_CLUSTER_NAME=CertificateManager
      - CASSANDRA_DC=dc1
      - CASSANDRA_RACK=rack3
      - CASSANDRA_SEEDS=cassandra-seed
      - CASSANDRA_ENDPOINT_SNITCH=GossipingPropertyFileSnitch
      - MAX_HEAP_SIZE=1G
      - HEAP_NEWSIZE=200M
    volumes:
      - cassandra-node-2-data:/var/lib/cassandra
      - ./ssl/node3:/etc/cassandra/ssl
    networks:
      - cassandra-net

volumes:
  cassandra-seed-data:
  cassandra-node-1-data:
  cassandra-node-2-data:

networks:
  cassandra-net:
    driver: bridge
```

### Health Check Implementation
```bash
#!/bin/bash
# health-check.sh

set -e

# Check if Cassandra is running
if ! pgrep -x "java" > /dev/null; then
    echo "Cassandra process not running"
    exit 1
fi

# Check CQL connectivity
if ! cqlsh -e "SELECT now() FROM system.local" > /dev/null 2>&1; then
    echo "Cannot connect to Cassandra via CQL"
    exit 1
fi

# Check node status
STATUS=$(nodetool status | grep "^UN" | wc -l)
if [ "$STATUS" -eq 0 ]; then
    echo "Node is not in UN (Up/Normal) state"
    exit 1
fi

echo "Cassandra is healthy"
exit 0
```

## Configuration Management

### Environment Variables
- `CASSANDRA_CLUSTER_NAME`: Cluster identifier
- `CASSANDRA_DC`: Datacenter name
- `CASSANDRA_RACK`: Rack identifier
- `CASSANDRA_SEEDS`: Comma-separated seed nodes
- `CASSANDRA_SSL_ENABLED`: Enable SSL (true/false)
- `CASSANDRA_AUTH_ENABLED`: Enable authentication (true/false)

### Volume Mounts
- `/var/lib/cassandra`: Data directory
- `/etc/cassandra/ssl`: SSL certificates and keystores
- `/etc/cassandra/triggers`: Custom triggers
- `/var/log/cassandra`: Log files

## Testing Requirements
1. **Unit Tests**
   - Container build verification
   - Configuration validation
   - Health check script testing

2. **Integration Tests**
   - Cluster formation testing
   - Data persistence verification
   - Network partition resilience

3. **Performance Tests**
   - Container startup time
   - Memory usage optimisation
   - I/O performance benchmarking

## Definition of Done
- [ ] Dockerfile builds successfully
- [ ] Docker Compose brings up multi-node cluster
- [ ] Health checks pass consistently
- [ ] Data persists across container restarts
- [ ] SSL/TLS encryption works between nodes
- [ ] Documentation includes deployment guide
- [ ] Automated tests achieve 90% coverage
- [ ] Security scan shows no vulnerabilities