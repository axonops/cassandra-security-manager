# User Story 10.1 - Docker Image Creation

## User Story
**As a** DevOps engineer  
**I want** production-ready Docker images for the Cassandra Security Manager  
**So that** I can easily deploy and scale the application in containerized environments

## INVEST Criteria
- **Independent**: Container creation separate from application development
- **Negotiable**: Base images and Optimisation strategies are flexible
- **Valuable**: Enables modern deployment patterns and cloud adoption
- **Estimable**: 1 sprint for complete containerization solution
- **Small**: Focused on containerization and distribution only
- **Testable**: Container functionality and security can be validated

## Acceptance Criteria

### Scenario: Multi-Stage Docker Build
```gherkin
Given efficient and secure container images are required
When building the Docker image
Then implement multi-stage build:
  | Build Stage            | Purpose                                 |
  | Base dependencies     | Install system dependencies             |
  | Python dependencies   | Install Python packages                 |
  | Application build     | Copy application code                   |
  | Runtime image         | Minimal runtime environment             |
  | Security hardening    | Remove unnecessary components           |
And Optimise for:
  - Layer caching efficiency
  - Minimal final image size
  - Security vulnerability reduction
  - Build reproducibility
  - Development vs production variants
```

### Scenario: Security-Hardened Base Image
```gherkin
Given container security is paramount
When selecting base image
Then use security-hardened approach:
  | Image Type             | Recommendation                          |
  | Base image            | python:3.11-slim or distroless          |
  | User context          | Non-root user (app:app)                 |
  | Filesystem            | Read-only root filesystem               |
  | Capabilities          | Drop all unnecessary capabilities       |
  | Security scanning     | Continuous vulnerability scanning       |
And implement security measures:
  - Remove package managers from final image
  - Use specific version tags (never :latest)
  - Sign images with content trust
  - Implement resource limits
  - Use health checks for reliability
```

### Scenario: Configuration Management
```gherkin
Given containers need flexible configuration
When configuring the application
Then support configuration via:
  | Configuration Method   | Implementation                          |
  | Environment variables | All application settings configurable   |
  | Config files          | Volume-mounted configuration files      |
  | Secrets               | Secure secret injection mechanisms      |
  | Command line args     | Override specific settings              |
  | Config maps           | Kubernetes ConfigMap integration        |
And provide configuration examples:
  - Docker run commands
  - Docker Compose files
  - Kubernetes manifests
  - Environment variable documentation
  - Configuration validation
```

### Scenario: Health Check Integration
```gherkin
Given containers need health monitoring
When implementing health checks
Then provide comprehensive monitoring:
  | Health Check Type      | Implementation                          |
  | Container health      | Docker HEALTHCHECK instruction          |
  | Application readiness | HTTP endpoint for readiness probe       |
  | Application liveness  | HTTP endpoint for liveness probe        |
  | Startup probe         | HTTP endpoint for startup validation    |
  | Dependency health     | Check database and external services    |
And implement check endpoints:
  - /health - Overall application health
  - /health/ready - Readiness for traffic
  - /health/live - Application liveness
  - /metrics - Prometheus metrics
  - /info - Application information
```

### Scenario: Volume and Data Management
```gherkin
Given containers need persistent data handling
When configuring data storage
Then support:
  | Data Type              | Storage Strategy                        |
  | Application logs      | Stdout/stderr for container logging     |
  | Certificate storage   | Volume mount for persistent storage     |
  | Configuration files   | ConfigMap or volume mounts              |
  | Temporary files       | tmpfs for temporary data                |
  | Backup data           | External volume for backup storage      |
And provide volume examples:
  - Named volumes for persistence
  - Bind mounts for development
  - tmpfs for temporary data
  - NFS/distributed storage integration
  - Backup and restore procedures
```

### Scenario: Multi-Architecture Support
```gherkin
Given deployment targets vary by architecture
When building container images
Then support multiple architectures:
  | Architecture           | Platform                                |
  | amd64                 | Intel/AMD 64-bit (most common)         |
  | arm64                 | ARM 64-bit (Apple Silicon, ARM servers)|
  | arm/v7                | ARM 32-bit (Raspberry Pi)               |
And implement using:
  - Docker buildx for multi-platform builds
  - Automated build pipeline for all platforms
  - Platform-specific Optimisations
  - Architecture detection in deployment
  - Unified manifest for all architectures
```

### Scenario: Development vs Production Images
```gherkin
Given different environments have different needs
When building images for different purposes
Then provide image variants:
  | Image Variant          | Purpose                                 |
  | Production            | Optimised, minimal, secure              |
  | Development           | Development tools, debugging enabled    |
  | Debug                 | Shell access, troubleshooting tools     |
  | Testing               | Test frameworks and utilities           |
And implement differences:
  - Production: Minimal, no shell, no dev tools
  - Development: Include development dependencies
  - Debug: Include shell, debugging tools
  - Testing: Include test frameworks and data
```

### Scenario: Container Orchestration Integration
```gherkin
Given containers are deployed via orchestration platforms
When integrating with orchestration
Then provide deployment artifacts:
  | Platform              | Artifacts Provided                       |
  | Kubernetes            | Deployment, Service, ConfigMap, Secret   |
  | Docker Compose        | docker-compose.yml with all services     |
  | Docker Swarm          | Stack files with service definitions     |
  | Nomad                 | Job specifications with health checks    |
  | OpenShift             | DeploymentConfig and BuildConfig         |
And include:
  - Resource requests and limits
  - Security contexts
  - Network policies
  - Service mesh integration
  - Auto-scaling configuration
```

### Scenario: Security Scanning and Compliance
```gherkin
Given container security is continuously monitored
When implementing security scanning
Then integrate:
  | Security Tool          | Integration Point                       |
  | Trivy                 | Build-time vulnerability scanning       |
  | Snyk                  | Dependency vulnerability checking       |
  | Docker Scout          | Continuous security monitoring          |
  | Anchore               | Policy-based security scanning          |
  | Falco                 | Runtime security monitoring             |
And enforce policies:
  - Block builds with critical vulnerabilities
  - Regular security scans of deployed images
  - Compliance reporting (SOC2, PCI-DSS)
  - Security baseline validation
  - Incident response for security issues
```

### Scenario: Image Registry and Distribution
```gherkin
Given images need secure distribution
When distributing container images
Then support multiple registries:
  | Registry Type          | Implementation                          |
  | Docker Hub            | Public distribution for open source     |
  | Private registry      | Self-hosted enterprise distribution     |
  | Cloud registries      | AWS ECR, Azure ACR, Google GCR         |
  | Artifact registries   | OCI-compliant artifact storage          |
And implement security:
  - Image signing with Cosign
  - Vulnerability scanning before push
  - Access control and authentication
  - Image provenance tracking
  - Supply chain security validation
```

### Scenario: Resource Optimisation
```gherkin
Given container resources need Optimisation
When Optimising resource usage
Then implement:
  | Optimisation Area      | Technique                               |
  | Image size            | Multi-stage builds, distroless base     |
  | Startup time          | Application pre-warming, dependency caching|
  | Memory usage          | Optimised Python runtime settings       |
  | CPU usage             | Efficient application architecture       |
  | Network               | Connection pooling, keep-alive          |
And provide guidance:
  - Resource requirement recommendations
  - Scaling guidelines
  - Performance tuning parameters
  - Monitoring and alerting setup
  - Cost Optimisation strategies
```

## Edge Cases and Security Considerations

### Edge Case: Container Image Corruption
```gherkin
Given container images may be corrupted
When image corruption is detected
Then:
  - Implement image integrity verification
  - Provide automatic retry mechanisms
  - Use image digest verification
  - Implement fallback to previous versions
  - Monitor image pull success rates
```

### Edge Case: Registry Unavailability
```gherkin
Given container registries may be unavailable
When registry access fails
Then:
  - Implement local image caching
  - Support multiple registry mirrors
  - Provide offline deployment options
  - Cache images in edge locations
  - Implement graceful degradation
```

### Edge Case: Resource Constraints
```gherkin
Given deployment environments may have limited resources
When resources are constrained
Then:
  - Provide minimal resource configuration
  - Implement resource usage monitoring
  - Support horizontal pod autoscaling
  - Optimise for low-memory environments
  - Provide performance tuning guides
```

## Security Hardening

```gherkin
Given containers are potential attack vectors
Then implement security controls:
  | Security Control       | Implementation                          |
  | Non-root execution    | Run as non-privileged user              |
  | Read-only filesystem  | Prevent runtime file modifications      |
  | No package managers   | Remove apt, yum, pip from final image  |
  | Minimal attack surface| Include only necessary components       |
  | Security scanning     | Continuous vulnerability monitoring     |
  | Content trust         | Sign and verify image integrity         |
  | Resource limits       | Prevent resource exhaustion attacks     |
  | Network isolation     | Default deny network policies           |
```

## Performance Requirements

```gherkin
Given container performance affects deployment speed
Then meet these targets:
  | Performance Metric     | Target                                  |
  | Image build time      | < 5 minutes for complete build          |
  | Image size            | < 500MB for production image            |
  | Container startup     | < 30 seconds to ready state             |
  | Health check response | < 100ms for health endpoints            |
  | Memory usage          | < 512MB baseline memory usage           |
  | CPU efficiency        | < 0.5 CPU cores at idle                 |
```

## Dockerfile Example

```dockerfile
# Multi-stage build for Cassandra Security Manager
FROM python:3.11-slim as builder

# Install build dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Production image
FROM python:3.11-slim

# Create non-root user
RUN groupadd -r app && useradd -r -g app app

# Install runtime dependencies only
RUN apt-get update && apt-get install -y \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get autoremove -y

# Copy dependencies from builder
COPY --from=builder /root/.local /home/app/.local

# Copy application
COPY --chown=app:app . /app
WORKDIR /app

# Switch to non-root user
USER app

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8000/health')"

# Expose port
EXPOSE 8000

# Start application
CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

## Technical Notes
- Use Docker BuildKit for enhanced build features
- Implement .dockerignore for build efficiency
- Use multi-stage builds for size Optimisation
- Implement proper signal handling in containers
- Use init system for proper process management
- Implement graceful shutdown handling
- Plan for image scanning integration
- Consider using scratch or distroless base images

## Definition of Done
- [ ] Multi-stage Docker build implemented
- [ ] Security-hardened base image selected
- [ ] Configuration management via environment variables
- [ ] Health check endpoints implemented
- [ ] Volume and data management configured
- [ ] Multi-architecture support enabled
- [ ] Development and production image variants
- [ ] Container orchestration integration provided
- [ ] Security scanning pipeline integrated
- [ ] Image registry and distribution configured
- [ ] Resource Optimisation implemented
- [ ] Edge cases handled
- [ ] Performance targets met
- [ ] Security hardening applied
- [ ] Documentation and examples complete
- [ ] Container testing automated