# Contributing to Cassandra Security Manager

Thank you for your interest in contributing to Cassandra Security Manager! This document provides guidelines and instructions for contributing to the project.

## Code of Conduct

By participating in this project, you agree to abide by our [Code of Conduct](CODE_OF_CONDUCT.md). Please read it before contributing.

## How to Contribute

### Reporting Issues

1. **Check existing issues** to avoid duplicates
2. **Use issue templates** when available
3. **Provide detailed information**:
   - Version of Cassandra Security Manager
   - Cassandra version
   - Python version
   - Steps to reproduce
   - Expected vs actual behavior
   - Error messages and stack traces

### Suggesting Features

1. **Check the roadmap** in docs/requirements/
2. **Open a discussion** before creating a feature request
3. **Provide use cases** and business value
4. **Consider security implications**

### Contributing Code

#### Prerequisites

- Python 3.11 or higher
- Docker for running Cassandra tests
- Git with commit signing configured
- Understanding of certificate management and cryptography

#### Development Setup

1. **Fork the repository**
   ```bash
   git clone https://github.com/YOUR_USERNAME/cassandra-security-manager.git
   cd cassandra-security-manager
   ```

2. **Create a virtual environment**
   ```bash
   python3.11 -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   ```

3. **Install dependencies**
   ```bash
   pip install -r requirements.txt
   pip install -r requirements-dev.txt
   ```

4. **Run tests to verify setup**
   ```bash
   pytest --cov=. --cov-fail-under=90
   ```

#### Development Process

1. **Create a feature branch**
   ```bash
   git checkout -b feature/your-feature-name
   ```

2. **Write production-ready code**
   - No placeholders or TODOs
   - No mock objects in tests
   - Self-documenting code
   - Type hints for all functions

3. **Write comprehensive tests**
   - Unit tests for all functions
   - Integration tests against real Cassandra
   - Acceptance tests for user stories
   - Maintain 90% code coverage minimum

4. **Update documentation**
   - API documentation
   - User guides
   - Architecture decisions
   - Configuration examples

5. **Run quality checks**
   ```bash
   # Format code
   black .
   isort .
   
   # Type checking
   mypy .
   
   # Linting
   flake8 .
   pylint api core
   
   # Security scanning
   bandit -r api core
   safety check
   
   # Tests with coverage
   pytest --cov=. --cov-report=html --cov-fail-under=90
   ```

6. **Commit with clear messages**
   ```bash
   git commit -s -m "feat: add support for PKCS#11 HSM integration
   
   - Implement PKCS#11 interface for HSM communication
   - Add configuration for supported HSM vendors
   - Create abstraction layer for key operations
   - Update certificate generation to use HSM keys
   
   Closes #123"
   ```

#### Pull Request Guidelines

1. **Before submitting**:
   - Rebase on latest main branch
   - Ensure all tests pass
   - Update documentation
   - Add entry to CHANGELOG.md

2. **PR description must include**:
   - Problem being solved
   - Solution approach
   - Testing performed
   - Security considerations
   - Performance impact
   - Breaking changes (if any)

3. **PR requirements**:
   - All CI checks must pass
   - Code coverage ≥ 90%
   - No security vulnerabilities
   - Performance benchmarks for critical paths
   - Two approvals from maintainers

### Testing Standards

#### Test Categories

1. **Unit Tests** (`tests/unit/`)
   - Test individual functions/methods
   - Use pytest fixtures
   - Mock only external services (not Cassandra)

2. **Integration Tests** (`tests/integration/`)
   - Test component interactions
   - Use real Cassandra in Docker
   - Test error scenarios

3. **Acceptance Tests** (`tests/acceptance/`)
   - Test complete user stories
   - Use Gherkin scenarios from requirements
   - End-to-end workflows

#### Example Test Structure

```python
import pytest
from testcontainers.cassandra import CassandraContainer

from api.services.certificate import CertificateService
from core.crypto import KeyGenerator


class TestCertificateGeneration:
    @pytest.fixture(scope="class")
    def cassandra(self):
        with CassandraContainer("cassandra:4.1") as container:
            container.start()
            yield container
    
    @pytest.fixture
    def certificate_service(self, cassandra):
        return CertificateService(cassandra.get_connection_url())
    
    async def test_generate_rsa_certificate(self, certificate_service):
        request = CertificateRequest(
            common_name="test.example.com",
            algorithm="RSA",
            key_size=4096,
            validity_days=365
        )
        
        certificate = await certificate_service.generate(request)
        
        assert certificate.algorithm == "RSA"
        assert certificate.key_size == 4096
        assert certificate.is_valid()
```

### Documentation Standards

1. **Code Documentation**:
   - Docstrings for all public functions/classes
   - Type hints for all parameters and returns
   - Examples in docstrings for complex functions

2. **API Documentation**:
   - OpenAPI/Swagger annotations
   - Request/response examples
   - Error scenarios
   - Authentication requirements

3. **User Documentation**:
   - Getting started guide
   - Configuration reference
   - Deployment guides
   - Troubleshooting section

### Security Considerations

All contributions must consider security implications:

1. **Cryptographic Code**:
   - Use established libraries (cryptography.io)
   - Never implement custom crypto
   - Follow NIST guidelines
   - Support algorithm agility

2. **Input Validation**:
   - Validate all user inputs
   - Use Pydantic models
   - Implement rate limiting
   - Prevent injection attacks

3. **Secret Management**:
   - Never log secrets
   - Use secure key storage
   - Implement key rotation
   - Zero-knowledge principles

### Performance Guidelines

1. **Database Operations**:
   - Use prepared statements
   - Implement connection pooling
   - Batch operations when possible
   - Monitor query performance

2. **API Performance**:
   - Target < 100ms response time
   - Implement caching where appropriate
   - Use async operations
   - Profile critical paths

3. **Resource Usage**:
   - Monitor memory usage
   - Implement circuit breakers
   - Handle backpressure
   - Clean up resources

## Governance

### Maintainers

- Review and merge pull requests
- Make architectural decisions
- Handle security issues
- Plan releases

### Contributors

- Submit pull requests
- Review others' code
- Participate in discussions
- Help with documentation

### Decision Making

1. **Minor changes**: One maintainer approval
2. **Major changes**: Two maintainer approvals
3. **Architecture changes**: Team consensus
4. **Security changes**: Security review required

## Release Process

1. **Version numbering**: Semantic versioning (MAJOR.MINOR.PATCH)
2. **Release branches**: Created from main
3. **Release notes**: Generated from CHANGELOG.md
4. **Security advisories**: Published when needed
5. **Docker images**: Tagged with version

## Getting Help

- **Discord**: [Join our server](https://discord.gg/cassandra-security-manager)
- **Discussions**: GitHub Discussions for questions
- **Issues**: GitHub Issues for bugs
- **Security**: cassandra-security-manager-security@googlegroups.com for vulnerabilities

## Recognition

Contributors are recognized in:
- CONTRIBUTORS.md file
- Release notes
- Project documentation
- Annual contributor report

## License

By contributing to Cassandra Security Manager, you agree that your contributions will be licensed under the [Apache License 2.0](LICENSE.txt).

Thank you for contributing to Cassandra Security Manager!
