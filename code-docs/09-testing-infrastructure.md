# Testing Infrastructure

This document explains the testing infrastructure, test organization, and how to write effective tests for Guardrails.

## 🧪 Test Organization

### Test Structure

```
tests/
├── conftest.py                 # Shared fixtures and configuration
├── data/                       # Test data files
│   ├── sample.json
│   ├── test.rail
│   └── ...
├── unit_tests/                 # Fast, isolated tests
│   ├── test_guard.py
│   ├── test_validator.py
│   ├── test_runner.py
│   ├── cli/
│   │   ├── test_configure.py
│   │   └── test_start.py
│   └── ...
└── integration_tests/          # End-to-end tests
    ├── test_llm_integration.py
    ├── test_hub_integration.py
    └── ...
```

### Test Types

**Unit Tests:**
- Test individual components in isolation
- Fast execution (< 1 second per test)
- Mock external dependencies
- Located in `tests/unit_tests/`

**Integration Tests:**
- Test component interactions
- May call external APIs (with mocking)
- Slower execution
- Located in `tests/integration_tests/`

## ⚙️ Test Configuration

### pytest Configuration

**Location:** `pyproject.toml`

```toml
[tool.pytest.ini_options]
python_classes = ["Test"]
python_functions = ["test_"]
python_files = ["test_*.py"]
testpaths = ["tests"]
markers = [
    "no_hub_telemetry_mock"
]
```

### Running Tests

```bash
# All tests
pytest tests/

# Specific test file
pytest tests/unit_tests/test_guard.py

# Specific test
pytest tests/unit_tests/test_guard.py::TestGuard::test_validate

# With verbose output
pytest -v

# With output printed
pytest -s

# Stop on first failure
pytest -x

# Run last failed tests
pytest --lf

# Run tests matching pattern
pytest -k "test_validate"
```

### Test Coverage

```bash
# Run with coverage
pytest tests/ --cov=./guardrails/

# Generate HTML report
pytest tests/ --cov=./guardrails/ --cov-report=html

# View report
open htmlcov/index.html

# Or use make
make test-cov
make view-test-cov
```

## 🔧 Common Fixtures

### conftest.py

**Location:** `tests/conftest.py`

Common fixtures available to all tests:

```python
import pytest
from guardrails import Guard
from pydantic import BaseModel


@pytest.fixture
def simple_guard():
    """Create a basic guard for testing."""
    return Guard()


@pytest.fixture
def sample_schema():
    """Return sample JSON schema."""
    return {
        "type": "object",
        "properties": {
            "name": {"type": "string"},
            "age": {"type": "integer"}
        },
        "required": ["name"]
    }


@pytest.fixture
def sample_pydantic_model():
    """Return sample Pydantic model."""
    class Person(BaseModel):
        name: str
        age: int
    
    return Person


@pytest.fixture
def mock_llm_api():
    """Mock LLM API that returns JSON."""
    from unittest.mock import Mock
    
    mock_api = Mock()
    mock_api.return_value = Mock(
        choices=[
            Mock(
                message=Mock(content='{"name": "John", "age": 30}')
            )
        ]
    )
    return mock_api


@pytest.fixture
def sample_validator():
    """Create a sample validator."""
    from guardrails import Validator, register_validator
    from guardrails.classes import PassResult
    
    @register_validator(name="test_validator", data_type="string")
    class TestValidator(Validator):
        def validate(self, value, metadata):
            return PassResult()
    
    return TestValidator
```

### Using Fixtures

```python
def test_with_fixtures(simple_guard, sample_schema):
    """Test using fixtures."""
    guard = Guard.from_dict(sample_schema)
    assert guard is not None


def test_multiple_fixtures(simple_guard, mock_llm_api):
    """Test with multiple fixtures."""
    result = simple_guard(
        llm_api=mock_llm_api,
        prompt="test"
    )
    assert result is not None
```

## 🎭 Mocking Strategies

### Mocking LLM APIs

**Simple Mock:**
```python
from unittest.mock import Mock

def test_guard_with_mock():
    mock_llm = Mock()
    mock_llm.return_value = "Generated text"
    
    guard = Guard()
    result = guard(llm_api=mock_llm, prompt="test")
    
    assert mock_llm.called
```

**Mock with OpenAI Structure:**
```python
def test_openai_mock():
    from unittest.mock import Mock
    
    mock_response = Mock()
    mock_response.choices = [
        Mock(message=Mock(content='{"key": "value"}'))
    ]
    
    mock_api = Mock(return_value=mock_response)
    
    guard = Guard()
    result = guard(llm_api=mock_api, prompt="test")
```

**Using patch:**
```python
from unittest.mock import patch

@patch('openai.chat.completions.create')
def test_with_patch(mock_create):
    mock_create.return_value = Mock(
        choices=[Mock(message=Mock(content="test"))]
    )
    
    import openai
    guard = Guard()
    result = guard(
        llm_api=openai.chat.completions.create,
        prompt="test"
    )
    
    assert mock_create.called
```

### Mocking File System

```python
from unittest.mock import mock_open, patch

def test_file_read():
    mock_data = "file content"
    
    with patch("builtins.open", mock_open(read_data=mock_data)):
        # Code that reads file
        with open("test.txt") as f:
            content = f.read()
        
        assert content == mock_data
```

### Mocking Network Requests

```python
from unittest.mock import patch
import requests

@patch('requests.get')
def test_api_call(mock_get):
    mock_get.return_value.json.return_value = {"status": "ok"}
    mock_get.return_value.status_code = 200
    
    response = requests.get("https://api.example.com")
    assert response.json() == {"status": "ok"}
```

### Mocking Environment Variables

```python
from unittest.mock import patch
import os

@patch.dict(os.environ, {"GUARDRAILS_TOKEN": "test-token"})
def test_with_env_var():
    token = os.getenv("GUARDRAILS_TOKEN")
    assert token == "test-token"
```

## 📝 Test Patterns

### Testing Guard Validation

```python
class TestGuardValidation:
    """Test Guard validation methods."""
    
    def test_validate_success(self):
        """Test successful validation."""
        from guardrails import Guard
        from guardrails.hub import RegexMatch
        
        guard = Guard().use(
            RegexMatch(regex=r"\d+")
        )
        
        result = guard.validate("123")
        
        assert result.validation_passed
        assert result.validated_output == "123"
    
    def test_validate_failure(self):
        """Test failed validation."""
        guard = Guard().use(
            RegexMatch(regex=r"\d+")
        )
        
        result = guard.validate("abc")
        
        assert not result.validation_passed
        assert result.error is not None
    
    def test_validate_with_metadata(self):
        """Test validation with metadata."""
        guard = Guard()
        metadata = {"user_id": "123"}
        
        result = guard.validate("test", metadata=metadata)
        
        # Validator should have access to metadata
        assert result is not None
```

### Testing Validators

```python
class TestMyValidator:
    """Test custom validator."""
    
    def test_validator_pass(self):
        """Test validator passes."""
        from my_validator import MyValidator
        
        validator = MyValidator(param="value")
        result = validator.validate("valid input", {})
        
        assert result.passed
        assert result.value_override is None
    
    def test_validator_fail(self):
        """Test validator fails."""
        validator = MyValidator(param="value")
        result = validator.validate("invalid", {})
        
        assert not result.passed
        assert "error" in result.error_message.lower()
    
    def test_validator_fix(self):
        """Test validator suggests fix."""
        validator = MyValidator(param="value")
        result = validator.validate("fixable", {})
        
        assert not result.passed
        assert result.fix_value is not None
    
    def test_validator_serialization(self):
        """Test validator can be serialized."""
        validator = MyValidator(param="value")
        
        serialized = validator.to_dict()
        
        assert serialized["name"] == "my_validator"
        assert serialized["args"]["param"] == "value"
```

### Testing Async Code

```python
import pytest


@pytest.mark.asyncio
async def test_async_guard():
    """Test AsyncGuard."""
    from guardrails import AsyncGuard
    
    guard = AsyncGuard()
    result = await guard.validate("test")
    
    assert result.validation_passed


@pytest.mark.asyncio
async def test_async_with_mock():
    """Test async with mocked LLM."""
    from unittest.mock import AsyncMock
    
    mock_llm = AsyncMock()
    mock_llm.return_value = "response"
    
    guard = AsyncGuard()
    result = await guard(llm_api=mock_llm, prompt="test")
    
    assert mock_llm.called


@pytest.mark.asyncio
async def test_concurrent_validation():
    """Test concurrent validation."""
    import asyncio
    from guardrails import AsyncGuard
    
    guard = AsyncGuard()
    
    tasks = [
        guard.validate(f"test{i}")
        for i in range(10)
    ]
    
    results = await asyncio.gather(*tasks)
    
    assert len(results) == 10
    assert all(r.validation_passed for r in results)
```

### Testing CLI Commands

```python
from typer.testing import CliRunner
from guardrails.cli.guardrails import guardrails

runner = CliRunner()


class TestCLI:
    """Test CLI commands."""
    
    def test_version(self):
        """Test version command."""
        result = runner.invoke(guardrails, ["version"])
        
        assert result.exit_code == 0
        assert "Guardrails AI" in result.output
    
    def test_configure(self):
        """Test configure command."""
        # Mock user input
        result = runner.invoke(
            guardrails,
            ["configure"],
            input="test-token\ny\n"
        )
        
        assert result.exit_code == 0
    
    def test_hub_install(self):
        """Test hub install command."""
        with patch('guardrails.hub.install') as mock_install:
            result = runner.invoke(
                guardrails,
                ["hub", "install", "hub://org/validator"]
            )
            
            assert result.exit_code == 0
            mock_install.assert_called_once()
```

### Parametrized Tests

```python
@pytest.mark.parametrize("input,expected", [
    ("valid1", True),
    ("valid2", True),
    ("invalid", False),
])
def test_multiple_inputs(input, expected):
    """Test with multiple inputs."""
    validator = MyValidator()
    result = validator.validate(input, {})
    assert result.passed == expected


@pytest.mark.parametrize("schema,data,should_pass", [
    ({"type": "string"}, "test", True),
    ({"type": "integer"}, 42, True),
    ({"type": "string"}, 42, False),
])
def test_schema_validation(schema, data, should_pass):
    """Test schema validation."""
    guard = Guard.from_dict(schema)
    result = guard.validate(data)
    assert result.validation_passed == should_pass
```

## 🎯 Test Markers

### Custom Markers

```python
# Define in pyproject.toml
[tool.pytest.ini_options]
markers = [
    "slow: marks tests as slow",
    "integration: marks tests as integration tests",
    "requires_api: tests that need API access",
]

# Use in tests
@pytest.mark.slow
def test_slow_operation():
    """Slow test."""
    pass

@pytest.mark.integration
def test_integration():
    """Integration test."""
    pass

@pytest.mark.requires_api
def test_with_api():
    """Test requiring API."""
    pass
```

### Running Specific Markers

```bash
# Run only slow tests
pytest -m slow

# Run excluding slow tests
pytest -m "not slow"

# Run integration tests
pytest -m integration
```

## 📊 Coverage Goals

### Coverage Targets

- **Overall**: > 80%
- **Core modules** (guard.py, validator_base.py): > 90%
- **Utils**: > 75%
- **CLI**: > 70%

### Checking Coverage

```bash
# Generate coverage report
pytest tests/ --cov=./guardrails/ --cov-report=term-missing

# Show files with < 80% coverage
pytest tests/ --cov=./guardrails/ --cov-fail-under=80
```

### Excluding from Coverage

```python
# Exclude specific lines
def debug_function():  # pragma: no cover
    """This is only for debugging."""
    print("debug info")

# Exclude blocks
if TYPE_CHECKING:  # pragma: no cover
    from typing import ...
```

## 🔍 Debugging Tests

### Print Debugging

```python
def test_with_debug():
    """Test with debug output."""
    guard = Guard()
    print(f"Guard: {guard}")  # Will show with pytest -s
    
    result = guard.validate("test")
    print(f"Result: {result}")
    
    assert result.validation_passed
```

### Using Debugger

```python
def test_with_debugger():
    """Test with debugger."""
    guard = Guard()
    
    import pdb; pdb.set_trace()  # Breakpoint
    # Or: breakpoint()
    
    result = guard.validate("test")
    assert result.validation_passed
```

### Pytest Debugging

```bash
# Drop into debugger on failure
pytest --pdb

# Drop into debugger at start
pytest --trace

# With ipdb
pytest --pdbcls=IPython.terminal.debugger:TerminalPdb --pdb
```

## 🚀 Performance Testing

### Timing Tests

```python
import time

def test_performance():
    """Test performance."""
    guard = Guard()
    
    start = time.time()
    
    for i in range(100):
        guard.validate(f"test{i}")
    
    duration = time.time() - start
    
    assert duration < 1.0  # Should complete in < 1 second
```

### Benchmarking

```python
import pytest

@pytest.mark.benchmark
def test_benchmark(benchmark):
    """Benchmark validation."""
    guard = Guard().use(MyValidator())
    
    result = benchmark(guard.validate, "test")
    
    assert result.validation_passed
```

## 🧹 Test Cleanup

### Setup and Teardown

```python
class TestWithCleanup:
    """Test with setup/teardown."""
    
    def setup_method(self):
        """Run before each test."""
        self.guard = Guard()
        self.temp_file = "temp.txt"
    
    def teardown_method(self):
        """Run after each test."""
        if os.path.exists(self.temp_file):
            os.remove(self.temp_file)
    
    def test_something(self):
        """Test method."""
        assert self.guard is not None
```

### Using Context Managers

```python
def test_with_context():
    """Test using context manager."""
    from tempfile import TemporaryDirectory
    
    with TemporaryDirectory() as tmpdir:
        # Use temporary directory
        file_path = os.path.join(tmpdir, "test.txt")
        # ... test code
        # Directory automatically cleaned up
```

## 📚 Best Practices

### 1. Test Names Should Be Descriptive

```python
# Good
def test_guard_validates_string_with_regex_match():
    pass

# Bad
def test_1():
    pass
```

### 2. One Assertion Per Test (Generally)

```python
# Good
def test_validation_passes():
    result = guard.validate("test")
    assert result.validation_passed

def test_validation_returns_correct_output():
    result = guard.validate("test")
    assert result.validated_output == "test"

# Acceptable for related assertions
def test_validation_result():
    result = guard.validate("test")
    assert result.validation_passed
    assert result.error is None
```

### 3. Use Fixtures for Common Setup

```python
# Good
@pytest.fixture
def configured_guard():
    return Guard().use(MyValidator())

def test_with_fixture(configured_guard):
    result = configured_guard.validate("test")
    assert result.validation_passed
```

### 4. Mock External Dependencies

```python
# Good - mocked
@patch('requests.get')
def test_with_mock(mock_get):
    mock_get.return_value.json.return_value = {}
    # test code

# Bad - actual API call
def test_without_mock():
    response = requests.get("https://api.example.com")
    # May fail if API is down
```

### 5. Test Edge Cases

```python
def test_empty_input():
    """Test with empty input."""
    result = validator.validate("", {})
    assert not result.passed

def test_none_input():
    """Test with None input."""
    result = validator.validate(None, {})
    assert not result.passed

def test_very_large_input():
    """Test with large input."""
    large_input = "x" * 1000000
    result = validator.validate(large_input, {})
    # Should handle gracefully
```

## 🎓 Learning Resources

### Pytest Documentation
- https://docs.pytest.org/

### Mocking Guide
- https://docs.python.org/3/library/unittest.mock.html

### Testing Best Practices
- https://testdriven.io/blog/testing-best-practices/

## 📚 Next Steps

- [Development Guide](./08-development-guide.md) - Development workflow
- [Module Reference](./07-module-reference.md) - Understanding modules
- [Contributing Guide](../CONTRIBUTING.md) - How to contribute

---

Good tests make for reliable, maintainable code. Happy testing! 🧪
