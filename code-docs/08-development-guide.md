# Development Guide

This guide explains how to set up your development environment, contribute code, and follow best practices when working on Guardrails.

## 🚀 Getting Started

### Prerequisites

- Python 3.9 or higher
- Git
- Poetry (optional, but recommended)

### Clone and Setup

```bash
# Clone the repository
git clone https://github.com/guardrails-ai/guardrails.git
cd guardrails

# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install development dependencies
make dev

# Or manually with pip
pip install -e ".[dev]"

# Install pre-commit hooks
pre-commit install
```

### Verify Installation

```bash
# Run basic tests
make test-basic

# Check imports work
python -c "import guardrails as gd; print(gd.__version__)"
```

## 🛠️ Development Workflow

### 1. Create a Branch

```bash
# Create feature branch
git checkout -b feature/my-feature

# Or bug fix branch
git checkout -b fix/issue-123
```

### 2. Make Changes

Edit code following the [Code Style](#code-style) guidelines.

### 3. Run Tests

```bash
# Run all tests
make test

# Run specific test file
pytest tests/unit_tests/test_guard.py

# Run with coverage
make test-cov

# View coverage report
make view-test-cov
```

### 4. Format Code

```bash
# Auto-format
make autoformat

# Or manually
ruff check guardrails/ tests/ --fix
ruff format guardrails/ tests/
docformatter --in-place --recursive guardrails tests
```

### 5. Type Check

```bash
# Run type checker
make type

# Or manually
pyright guardrails/
```

### 6. Lint Code

```bash
# Run linter
make lint

# Or manually
ruff check guardrails/ tests/
ruff format guardrails/ tests/ --check
```

### 7. Commit Changes

```bash
# Stage changes
git add .

# Commit (pre-commit hooks will run)
git commit -m "feat: add new feature"

# Or skip hooks (not recommended)
git commit --no-verify -m "..."
```

### 8. Push and Create PR

```bash
# Push branch
git push origin feature/my-feature

# Create pull request on GitHub
# Fill out PR template
# Request review
```

## 📝 Code Style

### Python Style

Guardrails follows PEP 8 with some modifications:

- **Line length**: 88 characters (Black default)
- **Quotes**: Double quotes for strings
- **Imports**: Sorted with isort
- **Type hints**: Required for public APIs

**Example:**
```python
from typing import Any, Dict, Optional

from guardrails import Validator
from guardrails.classes import PassResult, FailResult


class MyValidator(Validator):
    """
    Short description.
    
    Longer description with details about what this validator does.
    
    Args:
        param1: Description of param1
        param2: Description of param2
    """
    
    def __init__(
        self,
        param1: str,
        param2: int = 0,
        on_fail: Optional[str] = None
    ):
        super().__init__(on_fail=on_fail)
        self.param1 = param1
        self.param2 = param2
    
    def validate(self, value: Any, metadata: Dict[str, Any]) -> ValidationResult:
        """
        Validate the value.
        
        Args:
            value: Value to validate
            metadata: Additional context
            
        Returns:
            PassResult or FailResult
        """
        if self._is_valid(value):
            return PassResult()
        
        return FailResult(
            error_message=f"Invalid value: {value}"
        )
    
    def _is_valid(self, value: Any) -> bool:
        """Private helper method."""
        return True
```

### Documentation Style

Use Google-style docstrings:

```python
def my_function(arg1: str, arg2: int) -> bool:
    """
    Short description.
    
    Longer description with more details.
    
    Args:
        arg1: Description of arg1
        arg2: Description of arg2
    
    Returns:
        Description of return value
    
    Raises:
        ValueError: When invalid input
    
    Example:
        >>> my_function("test", 42)
        True
    """
    pass
```

### Import Organization

```python
# 1. Standard library imports
import os
import sys
from typing import Any, Dict

# 2. Third-party imports
import requests
from pydantic import BaseModel

# 3. Guardrails imports
from guardrails import Guard, Validator
from guardrails.classes import PassResult
from guardrails.utils import parse_llm_output
```

### Naming Conventions

- **Classes**: `PascalCase`
- **Functions/Methods**: `snake_case`
- **Constants**: `UPPER_CASE`
- **Private**: `_leading_underscore`
- **Protected**: `_single_underscore`
- **Dunder**: `__double_underscore__`

## 🧪 Testing

### Test Structure

```
tests/
├── unit_tests/           # Fast, isolated tests
│   ├── test_guard.py
│   ├── test_validator.py
│   └── ...
├── integration_tests/    # End-to-end tests
│   ├── test_llm_integration.py
│   └── ...
├── conftest.py           # Shared fixtures
└── data/                 # Test data files
```

### Writing Unit Tests

```python
# tests/unit_tests/test_my_validator.py
import pytest
from guardrails import Guard
from guardrails.hub import MyValidator


class TestMyValidator:
    """Test suite for MyValidator."""
    
    def test_valid_input(self):
        """Test validator passes with valid input."""
        validator = MyValidator(param="value")
        result = validator.validate("valid data", {})
        
        assert result.passed
        assert result.value_override is None
    
    def test_invalid_input(self):
        """Test validator fails with invalid input."""
        validator = MyValidator(param="value")
        result = validator.validate("invalid", {})
        
        assert not result.passed
        assert "Invalid" in result.error_message
    
    def test_with_guard(self):
        """Test validator works in Guard."""
        guard = Guard().use(MyValidator(param="value"))
        outcome = guard.validate("test")
        
        assert outcome.validation_passed
    
    @pytest.mark.parametrize("input,expected", [
        ("valid1", True),
        ("valid2", True),
        ("invalid", False),
    ])
    def test_multiple_inputs(self, input, expected):
        """Test validator with multiple inputs."""
        validator = MyValidator()
        result = validator.validate(input, {})
        assert result.passed == expected
```

### Using Fixtures

```python
# tests/conftest.py
import pytest
from guardrails import Guard


@pytest.fixture
def simple_guard():
    """Create a simple guard for testing."""
    return Guard()


@pytest.fixture
def sample_schema():
    """Return sample JSON schema."""
    return {
        "type": "object",
        "properties": {
            "name": {"type": "string"},
            "age": {"type": "integer"}
        }
    }


# Use in tests
def test_with_fixture(simple_guard):
    guard = simple_guard.use(MyValidator())
    assert guard is not None
```

### Mocking LLM Calls

```python
from unittest.mock import Mock, patch


def test_guard_with_llm():
    """Test Guard with mocked LLM."""
    # Mock LLM API
    mock_llm = Mock()
    mock_llm.return_value = Mock(
        choices=[Mock(message=Mock(content='{"name": "John"}'))]
    )
    
    guard = Guard.for_pydantic(Person)
    result = guard(
        llm_api=mock_llm,
        prompt="Generate person"
    )
    
    assert result.validated_output.name == "John"
    mock_llm.assert_called_once()


@patch('openai.chat.completions.create')
def test_with_patch(mock_create):
    """Test with patched OpenAI."""
    mock_create.return_value = Mock(
        choices=[Mock(message=Mock(content="test"))]
    )
    
    guard = Guard()
    result = guard(
        llm_api=openai.chat.completions.create,
        prompt="test"
    )
    
    assert mock_create.called
```

### Testing Async Code

```python
import pytest


@pytest.mark.asyncio
async def test_async_guard():
    """Test AsyncGuard."""
    from guardrails import AsyncGuard
    
    guard = AsyncGuard().use(MyValidator())
    result = await guard.validate("test")
    
    assert result.validation_passed


@pytest.mark.asyncio
async def test_async_validator():
    """Test async validator."""
    validator = MyAsyncValidator()
    result = await validator.validate_async("test", {})
    
    assert result.passed
```

## 🔧 Common Development Tasks

### Adding a New Validator

1. **Create validator file:**
```python
# guardrails/validators/my_validator.py
from guardrails import Validator, register_validator
from guardrails.classes import PassResult, FailResult


@register_validator(name="my_validator", data_type="string")
class MyValidator(Validator):
    """Validator description."""
    
    def __init__(self, param: str, **kwargs):
        super().__init__(**kwargs)
        self.param = param
    
    def validate(self, value, metadata):
        # Validation logic
        return PassResult()
```

2. **Add tests:**
```python
# tests/unit_tests/test_my_validator.py
class TestMyValidator:
    def test_validation(self):
        validator = MyValidator(param="test")
        result = validator.validate("value", {})
        assert result.passed
```

3. **Update exports:**
```python
# guardrails/validators/__init__.py
from guardrails.validators.my_validator import MyValidator

__all__ = ["MyValidator", ...]
```

4. **Document validator:**
- Add docstring with examples
- Update relevant documentation

### Adding a New CLI Command

1. **Create command file:**
```python
# guardrails/cli/my_command.py
import typer
from guardrails.cli.guardrails import guardrails


@guardrails.command()
def my_command(
    arg: str = typer.Argument(..., help="Required arg"),
    option: bool = typer.Option(False, help="Optional flag")
):
    """Command description."""
    # Implementation
    typer.echo(f"Running with {arg}")
```

2. **Import in CLI:**
```python
# guardrails/cli/__init__.py
from guardrails.cli.my_command import my_command
```

3. **Add tests:**
```python
# tests/unit_tests/cli/test_my_command.py
from typer.testing import CliRunner
from guardrails.cli.guardrails import guardrails

runner = CliRunner()

def test_my_command():
    result = runner.invoke(guardrails, ["my-command", "test"])
    assert result.exit_code == 0
```

### Modifying Guard Behavior

1. **Locate relevant code:** Usually in `guard.py` or `run/runner.py`
2. **Make changes**
3. **Add/update tests**
4. **Check backwards compatibility**
5. **Update documentation**

### Adding Dependencies

```bash
# Add runtime dependency
poetry add package-name

# Add dev dependency
poetry add --dev package-name

# Update lockfile
poetry lock --no-update

# Or with pip (update pyproject.toml)
pip install package-name
```

## 🐛 Debugging

### Using Debugger

```python
# Add breakpoint
import pdb; pdb.set_trace()

# Or with Python 3.7+
breakpoint()
```

### Enable Debug Logging

```python
# In code
from guardrails import configure_logging
import logging

configure_logging(level=logging.DEBUG)

# Via environment
export GUARDRAILS_LOG_LEVEL=DEBUG
```

### Debug Tests

```bash
# Run single test with output
pytest tests/unit_tests/test_guard.py::test_validate -v -s

# Drop into debugger on failure
pytest --pdb

# Drop into debugger on error
pytest --pdbcls=IPython.terminal.debugger:TerminalPdb --pdb
```

## 📊 Performance Profiling

### Profile Code

```python
import cProfile
import pstats

profiler = cProfile.Profile()
profiler.enable()

# Code to profile
guard = Guard().use(validator)
result = guard.validate(data)

profiler.disable()
stats = pstats.Stats(profiler)
stats.sort_stats('cumulative')
stats.print_stats(20)
```

### Memory Profiling

```python
from memory_profiler import profile

@profile
def test_memory():
    guard = Guard()
    # ... code to profile
```

## 🔄 Pull Request Process

### PR Checklist

- [ ] Code follows style guidelines
- [ ] All tests pass
- [ ] New tests added for new features
- [ ] Documentation updated
- [ ] Type hints added
- [ ] Backwards compatible (or documented)
- [ ] No merge conflicts
- [ ] PR description is clear

### PR Template

```markdown
## Description
Brief description of changes

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Testing
How was this tested?

## Checklist
- [ ] Tests pass
- [ ] Code formatted
- [ ] Documentation updated
```

### Review Process

1. **Submit PR** with clear description
2. **CI checks** must pass
3. **Code review** by maintainers
4. **Address feedback**
5. **Approval** and merge

## 📚 Documentation

### Update User Documentation

User-facing docs are in `docs/` (Docusaurus):

```bash
# Install dependencies
cd docs/
npm install

# Serve locally
npm run start

# Build
npm run build
```

### Update Code Documentation

Internal docs are in `code-docs/`:

```bash
# Edit markdown files
vim code-docs/01-architecture-overview.md

# Commit changes
git add code-docs/
git commit -m "docs: update architecture docs"
```

### API Documentation

```python
# Good docstring example
def my_function(arg1: str, arg2: int = 0) -> bool:
    """
    Short one-line description.
    
    Longer description with more context about what this
    function does and why it exists.
    
    Args:
        arg1: Description of first argument
        arg2: Description of second argument. Defaults to 0.
    
    Returns:
        True if successful, False otherwise
    
    Raises:
        ValueError: If arg1 is empty
        RuntimeError: If operation fails
    
    Example:
        >>> my_function("test", 42)
        True
    """
    if not arg1:
        raise ValueError("arg1 cannot be empty")
    
    return True
```

## 🚢 Release Process

### Version Bumping

```bash
# Update version in pyproject.toml
# Update CHANGELOG.md

# Commit
git add pyproject.toml CHANGELOG.md
git commit -m "chore: bump version to 0.6.8"

# Tag
git tag v0.6.8
git push origin v0.6.8
```

### Building Package

```bash
# Build
poetry build

# Check package
twine check dist/*

# Test upload
twine upload --repository testpypi dist/*

# Production upload
twine upload dist/*
```

## 🎯 Best Practices

### 1. Write Tests First (TDD)

```python
# 1. Write failing test
def test_new_feature():
    result = new_feature()
    assert result == expected

# 2. Implement feature
def new_feature():
    return expected

# 3. Refactor
```

### 2. Keep Functions Small

- Single responsibility
- Maximum ~50 lines
- Extract helpers

### 3. Use Type Hints

```python
from typing import Any, Dict, List, Optional

def process(data: List[Dict[str, Any]]) -> Optional[str]:
    pass
```

### 4. Handle Errors Gracefully

```python
try:
    result = risky_operation()
except SpecificError as e:
    logger.error(f"Operation failed: {e}")
    raise UserFacingException("User-friendly message")
```

### 5. Log Appropriately

```python
logger.debug("Detailed debug info")
logger.info("Normal operation")
logger.warning("Something unusual")
logger.error("Error occurred", exc_info=True)
```

## 🔗 Useful Commands

```bash
# Development
make dev              # Install dev dependencies
make autoformat       # Format code
make type            # Type check
make lint            # Lint code
make test            # Run tests
make test-cov        # Run with coverage

# Documentation
make docs-serve      # Serve docs locally
make docs-deploy     # Deploy docs

# Cleanup
make clean           # Remove build artifacts
rm -rf .venv         # Remove virtual environment
```

## 📚 Resources

- [Contributing Guide](../CONTRIBUTING.md)
- [Code of Conduct](../CODE_OF_CONDUCT.md)
- [GitHub Issues](https://github.com/guardrails-ai/guardrails/issues)
- [Discord Community](https://discord.gg/gw4cR9QvYE)

## 💡 Getting Help

- **Discord**: Ask in #development channel
- **GitHub Discussions**: For longer discussions
- **GitHub Issues**: For bugs and features
- **Code Review**: Tag maintainers in PRs

---

Happy coding! Thank you for contributing to Guardrails! 🎉
