# Testing NexusMCP

This guide explains how to run the tests and what to expect.

## Setup

```bash
# Install the package with dev dependencies
pip install -e ".[dev]"

# You'll need an API key to test actual LLM calls
export ANTHROPIC_API_KEY="sk-your-key-here"
```

## Running Tests

### Quick Test (mocked, no API key needed)
```bash
# Run only unit tests (mocked, no external dependencies)
pytest tests/test_agent.py tests/test_skills.py -v
```

### Full Test Suite (includes integration tests)
```bash
# Run all tests including integration tests
pytest tests/ -v

# Run with coverage report
pytest tests/ --cov=src --cov-report=html

# Run a specific test
pytest tests/test_integration.py::test_agent_full_run_with_mock_api -v
```

### Test Categories

| File | Type | Requires API Key | What it Tests |
|------|------|------------------|---------------|
| `test_agent.py` | Unit | No* | Agent class, lifecycle, config |
| `test_skills.py` | Unit | No | Skills registry, versioning |
| `test_integration.py` | Integration | No** | Full agent runs with mocked API |

*Note: API key required for `test_agent_full_run_with_mock_api`, but this test can be skipped
**Integration tests use mocks, so they don't need a real API key

## Understanding the Tests

### Unit Tests (`test_agent.py`)
```python
@pytest.mark.asyncio
async def test_agent_initialization(agent_config):
    """Pure unit test - no dependencies, no I/O"""
    agent = Agent(agent_config)
    assert agent.config.name == "test-agent"
```

### Integration Tests (`test_integration.py`)
```python
@pytest.mark.asyncio
async def test_agent_full_run_with_mock_api(test_agent, mock_anthropic_response):
    """Integration test - mocks the API but tests full flow"""
    with patch.object(test_agent._tools, 'complete', ...) as mock:
        result = await test_agent.run("Test task")
        assert result.succeeded
```

## Expected Output

### Successful Run
```
tests/test_agent.py::test_agent_initialization PASSED
tests/test_agent.py::test_agent_from_config_dict PASSED
tests/test_skills.py::test_skill_decorator PASSED
tests/test_integration.py::test_agent_full_run_with_mock_api PASSED

======================== 12 passed in 0.45s ========================
```

## Troubleshooting

### Error: `ImportError: No module named 'mcp_agent_kit'`
**Solution:** Install the package in development mode:
```bash
pip install -e .
```

### Error: `ModuleNotFoundError: No module named 'anthropic'`
**Solution:** Install dependencies:
```bash
pip install -e ".[dev]"
```

### Error: `asyncio.TimeoutError` in tests
**Solution:** Increase timeout:
```bash
pytest tests/ --timeout=10
```

## Running Tests in CI/CD

The `.github/workflows/ci.yml` automatically runs:
```bash
ruff check src tests      # Linting
black --check src tests   # Format check
mypy src                  # Type checking
pytest tests/ -v          # Full test suite with coverage
```

## Test Coverage

After running tests with coverage:
```bash
pytest tests/ --cov=src --cov-report=html
open htmlcov/index.html  # View coverage report
```

Current coverage targets:
- `agents/`: ≥80%
- `skills/`: ≥85%
- `memory/`: ≥75%
- `orchestrator/`: ≥70%

## Writing New Tests

### Test Structure
```python
"""tests/test_my_feature.py"""

import pytest
from unittest.mock import AsyncMock, patch

@pytest.fixture
def my_resource():
    """Setup shared test resources"""
    return MyClass()

@pytest.mark.asyncio
async def test_async_behavior(my_resource):
    """Test async functions"""
    result = await my_resource.some_method()
    assert result is not None

def test_sync_behavior():
    """Test sync functions"""
    obj = MyClass()
    assert obj.property == "expected"
```

### Mocking External Calls
```python
@pytest.mark.asyncio
async def test_with_mocked_api():
    """Mock external API calls"""
    with patch('module.external_function', new_callable=AsyncMock) as mock:
        mock.return_value = {"key": "value"}
        result = await my_function()
        assert mock.called
```

## CI/CD Integration

Tests run automatically on:
- Push to `main` or `develop`
- Pull requests to `main` or `develop`

View results in GitHub Actions: `.github/workflows/ci.yml`

---

**Questions?** Open an issue or check the main README.
