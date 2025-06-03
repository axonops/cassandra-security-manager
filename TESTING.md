# Testing Guide for Cassandra Security Manager

## Overview

Cassandra Security Manager follows a comprehensive testing strategy to ensure reliability, security, and performance. We maintain a minimum of 90% test coverage and test against real Apache Cassandra instances rather than mocks.

## Testing Philosophy

* **Real Systems**: All tests run against ephemeral Cassandra 4.1+ Docker containers
* **No Mocks**: Integration tests use real Cassandra, Redis, and other dependencies
* **Production Ready**: Tests validate production scenarios and edge cases
* **Security First**: Every security feature must have corresponding security tests
* **Performance Aware**: Critical paths include performance benchmarks

## Test Structure

```
tests/
├── unit/                    # Fast, isolated component tests
├── integration/            # Tests with real dependencies
├── acceptance/             # Gherkin scenario tests
├── security/              # Security-specific test suites
├── performance/           # Performance benchmarks
└── fixtures/              # Shared test data and utilities
```

## Running Tests

### Quick Start

```bash
# Run all tests with coverage
pytest --cov=. --cov-report=html --cov-fail-under=90

# Run specific test categories
pytest tests/unit/
pytest tests/integration/
pytest tests/acceptance/

# Run with verbose output
pytest -v

# Run tests matching pattern
pytest -k "test_certificate_generation"

# Run with parallel execution
pytest -n auto
```

### Test Environment Setup

Tests automatically provision required infrastructure using testcontainers:

```python
import pytest
from testcontainers.cassandra import CassandraContainer
from testcontainers.redis import RedisContainer

@pytest.fixture(scope="session")
def cassandra():
    with CassandraContainer("cassandra:4.1") as container:
        container.start()
        yield container

@pytest.fixture(scope="session")
def redis():
    with RedisContainer("redis:7-alpine") as container:
        container.start()
        yield container
```

## Writing Tests

### Unit Tests

Unit tests focus on individual components in isolation:

```python
# tests/unit/test_certificate_validator.py
import pytest
from cassandra_security_manager.core.crypto import CertificateValidator
from cassandra_security_manager.core.exceptions import ValidationError

class TestCertificateValidator:
    def test_validates_minimum_key_size(self):
        validator = CertificateValidator()
        
        with pytest.raises(ValidationError) as exc:
            validator.validate_key_size("RSA", 2048)
        
        assert "Minimum RSA key size is 4096" in str(exc.value)
    
    def test_accepts_valid_key_size(self):
        validator = CertificateValidator()
        validator.validate_key_size("RSA", 4096)  # Should not raise
```

### Integration Tests

Integration tests verify interactions with real systems:

```python
# tests/integration/test_certificate_storage.py
import pytest
from cassandra_security_manager.core.database import CertificateStorage

@pytest.mark.integration
async def test_stores_and_retrieves_certificate(cassandra):
    storage = CertificateStorage(cassandra.get_connection_url())
    await storage.initialize()
    
    certificate = await generate_test_certificate()
    cert_id = await storage.store(certificate)
    
    retrieved = await storage.get(cert_id)
    assert retrieved.serial_number == certificate.serial_number
    assert retrieved.subject == certificate.subject
```

### Acceptance Tests

Acceptance tests validate user stories using Gherkin scenarios:

```python
# tests/acceptance/test_certificate_generation.py
import pytest
from pytest_bdd import scenarios, given, when, then

scenarios('../features/certificate_generation.feature')

@given('a CA hierarchy is initialized')
def ca_hierarchy(test_ca):
    return test_ca

@when('I request a server certificate for "cassandra.example.com"')
async def request_certificate(ca_hierarchy, certificate_service):
    return await certificate_service.generate_server_certificate(
        ca=ca_hierarchy,
        common_name="cassandra.example.com"
    )

@then('the certificate should be valid for TLS')
def verify_tls_certificate(certificate):
    assert certificate.has_extension("serverAuth")
    assert certificate.key_size >= 4096
```

### Security Tests

Security tests validate cryptographic operations and access controls:

```python
# tests/security/test_quantum_resistance.py
import pytest
from cassandra_security_manager.core.crypto import QuantumResistantSigner

@pytest.mark.security
async def test_dilithium_signature_verification():
    signer = QuantumResistantSigner(algorithm="dilithium3")
    
    key_pair = await signer.generate_key_pair()
    message = b"Critical security operation"
    
    signature = await signer.sign(message, key_pair.private_key)
    assert await signer.verify(message, signature, key_pair.public_key)
    
    # Ensure tampering is detected
    tampered = message + b"x"
    assert not await signer.verify(tampered, signature, key_pair.public_key)
```

### Performance Tests

Performance tests ensure operations meet SLA requirements:

```python
# tests/performance/test_certificate_generation_performance.py
import pytest
import asyncio
import time

@pytest.mark.performance
async def test_concurrent_certificate_generation(certificate_service, benchmark):
    async def generate_batch():
        tasks = [
            certificate_service.generate_certificate(f"node{i}.cluster.local")
            for i in range(100)
        ]
        return await asyncio.gather(*tasks)
    
    result = benchmark(generate_batch)
    
    # All certificates should be generated within 10 seconds
    assert benchmark.stats['mean'] < 10.0
    assert len(result) == 100
```

## Test Fixtures

Common fixtures are provided in `tests/conftest.py`:

```python
# tests/conftest.py
import pytest
from testcontainers.cassandra import CassandraContainer
from cassandra_security_manager.core import create_app

@pytest.fixture(scope="session")
def app():
    """FastAPI application instance"""
    return create_app(testing=True)

@pytest.fixture
async def client(app):
    """Async test client"""
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac

@pytest.fixture
async def authenticated_client(client, test_user):
    """Client with authentication headers"""
    token = await generate_test_token(test_user)
    client.headers["Authorization"] = f"Bearer {token}"
    yield client
```

## Coverage Requirements

* Minimum 90% overall coverage
* 100% coverage for security-critical code
* Coverage reports generated in `htmlcov/`
* CI/CD enforces coverage requirements

View coverage reports:
```bash
pytest --cov=. --cov-report=html
open htmlcov/index.html
```

## Continuous Integration

Tests run automatically on:
* Every pull request
* Pre-commit hooks (unit tests only)
* Nightly full test suite
* Release candidates

GitHub Actions workflow:
```yaml
name: Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      cassandra:
        image: cassandra:4.1
      redis:
        image: redis:7-alpine
    
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-dev.txt
      
      - name: Run tests
        run: |
          pytest --cov=. --cov-report=xml --cov-fail-under=90
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

## Best Practices

### Test Independence
* Each test must be independent and idempotent
* Use transactions or cleanup fixtures
* Never rely on test execution order

### Test Data
* Use factories for complex test objects
* Parameterize tests for multiple scenarios
* Keep test data minimal but realistic

### Async Testing
* Use `pytest-asyncio` for async tests
* Properly await all async operations
* Test both success and error paths

### Error Testing
* Test all error conditions
* Verify error messages are helpful
* Ensure proper cleanup on errors

## Debugging Tests

### Running Single Tests
```bash
# Run with print statements visible
pytest -s tests/unit/test_specific.py::test_function

# Run with debugger
pytest --pdb tests/unit/test_specific.py

# Run with detailed failure output
pytest -vv --tb=long
```

### Test Logs
```bash
# Enable debug logging
pytest --log-cli-level=DEBUG

# Capture logs in tests
def test_operation(caplog):
    with caplog.at_level(logging.DEBUG):
        perform_operation()
    assert "Expected log message" in caplog.text
```

## Security Testing Guidelines

### Certificate Validation
* Test certificate chain validation
* Verify expiration handling
* Test revocation checking
* Validate key usage constraints

### Access Control
* Test all permission boundaries
* Verify token expiration
* Test role-based access
* Validate audit logging

### Cryptographic Operations
* Test with known test vectors
* Verify algorithm parameters
* Test key rotation scenarios
* Validate secure random generation

## Contributing Tests

When contributing:
1. Write tests for all new features
2. Ensure tests pass locally
3. Maintain or improve coverage
4. Follow existing test patterns
5. Document complex test scenarios

## Troubleshooting

### Common Issues

**Container startup failures**
```bash
# Increase container startup timeout
export TESTCONTAINERS_RYUK_DISABLED=true
export TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE=/var/run/docker.sock
```

**Cassandra connection issues**
```bash
# Wait for Cassandra to be ready
pytest --cassandra-startup-timeout=60
```

**Async test failures**
```python
# Ensure proper event loop handling
@pytest.mark.asyncio
async def test_async_operation():
    await asyncio.sleep(0)  # Yield control
```

## Additional Resources

* [pytest documentation](https://docs.pytest.org/)
* [pytest-asyncio](https://pytest-asyncio.readthedocs.io/)
* [testcontainers-python](https://testcontainers-python.readthedocs.io/)
* [pytest-bdd](https://pytest-bdd.readthedocs.io/)