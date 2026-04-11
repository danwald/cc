# Python Project Setup

Standards and tooling conventions for Python projects.

## Runtime & Tools

| Tool | Purpose | Version |
|------|---------|---------|
| Python | Runtime | 3.12+ |
| uv | Package manager / runner | latest |
| ruff | Linting + formatting | latest |
| mypy | Type checking | latest |
| pytest | Testing | latest |

## Commands

```bash
uv run ruff check --fix .          # Lint + auto-fix
uv run ruff format .               # Format
uv run mypy                        # Type check
uv run pytest                      # Run all tests
uv run pytest -k "pattern"         # Run specific tests
```

For quick one-off Python scripts: `PYTHONPATH=$PWD python tests/test_file.py` (ensure correct CWD).

## Project Layout

**Only the `cli` component uses a `src/` directory.** For all other components (e.g. `agents-api`), code lives directly in the package folder (e.g. `agents_api/`). Follow the existing pattern for each component.

Use `pyproject.toml` (not `requirements.txt`) for all project metadata and tool configuration.

## pyproject.toml Configuration

```toml
[project]
name = "your-project"
version = "0.1.0"
requires-python = ">=3.12"

[tool.ruff]
line-length = 96
target-version = "py312"

[tool.ruff.format]
quote-style = "double"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM", "ASYNC"]

[tool.ruff.lint.isort]
known-first-party = ["your_package"]

[tool.mypy]
python_version = "3.12"
strict = true

[tool.pytest.ini_options]
asyncio_mode = "auto"
addopts = "--cov --cov-report=term-missing --cov-fail-under=80"
```

## Coding Standards

- **Python**: 3.12+, FastAPI, `async/await` preferred.
- **Formatting**: `ruff` enforces 96-char lines, double quotes, sorted imports. Use the project's configured linter — don't reformat manually.
- **Typing**: Strict (Pydantic v2 models preferred); `from __future__ import annotations` in every file.
- **Naming**: `snake_case` (functions/variables), `PascalCase` (classes), `SCREAMING_SNAKE` (constants).
- **Docstrings**: Google-style for public functions/classes.

## Error Handling

Define typed, hierarchical exceptions in `exceptions.py`:

```python
class DomainException(Exception):
    """Base exception for all domain errors."""

class EntityNotFoundError(DomainException):
    """Raised when a requested entity does not exist."""

class ValidationError(DomainException):
    """Raised when domain validation fails."""
```

Usage pattern:

```python
from __future__ import annotations

from your_package.exceptions import ValidationError


async def process_data(data: dict) -> Result:
    try:
        return result
    except KeyError as e:
        raise ValidationError(f"Missing required field: {e}") from e
```

Rules:
- Catch specific exceptions, not generic `Exception`.
- Use context managers for resources (database connections, file handles).
- For async code, use `try/finally` to ensure cleanup.

## Testing

### pytest (standard projects)

```bash
uv run pytest                               # Run all
uv run pytest -k "pattern"                 # Filter by name
uv run pytest --cov --cov-fail-under=80    # With coverage gate
```

### Ward (used in agents-api)

```bash
uv run pytest                                         # Run all ward tests
uv run pytest --search "pattern"                      # Filter (NOT -k or -p)
uv run test --fail-limit 1 --search "pattern"         # Stop on first failure
```

Use descriptive test names: `@test("Descriptive name of what is being tested")`.

**Do NOT mix pytest and ward syntax** — ward uses `@test()` decorator, not pytest fixtures/classes.

## Pre-commit Hooks

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.9.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.13.0
    hooks:
      - id: mypy
        additional_dependencies: [types-all]
```

Pre-commit order: **format → lint → typecheck → tests** (fast things first).

## CI (GitHub Actions)

```yaml
jobs:
  quality:
    steps:
      - uses: astral-sh/setup-uv@v5
      - run: uv run ruff format --check .
      - run: uv run ruff check .
      - run: uv run mypy
      - run: uv run pytest --cov --cov-report=xml
      - uses: codecov/codecov-action@v4
```

Enable Dependabot for weekly dependency updates.

## Security

- Run `uvx pip-audit` in CI for vulnerability scanning.
- Never commit `.env` or credentials.
- Enable Dependabot for automated dependency updates.

## Python Expressions (agents-api specific)

Evaluated using `simpleeval` in a sandboxed environment.

- Use `validate_py_expression()` from `agents_api.activities.task_steps.base_evaluate` for static checks (syntax, undefined names, safety).
- Expressions have access to `_` (current input) and standard library modules.

```bash
# Test an expression
PYTHONPATH=$PWD python -c "
from agents_api.activities.task_steps.base_evaluate import validate_py_expression
print(validate_py_expression('$ your_expr_here'))
"
```

```python
"$_['customer']['total_orders'] > 5"
"$len([x for x in _['items'] if x['category'] == 'electronics']) > 0"
```

In `task_to_spec`-converted tasks, `kind_` denotes step type. For `if_else` steps, the condition is in the `if_` field (aliased as `"if"`).

## Common Pitfalls

- **Wrong CWD or `PYTHONPATH`**: ensure you are in the correct directory (e.g. `agents-api/` not root for some `agents-api` tasks).
- **Mixing pytest & ward syntax**: ward uses `@test()` decorator, not pytest fixtures/classes.
- **Forgetting to activate venv**: run `source .venv/bin/activate` or use `uv run`.
- **`src/` layout confusion**: only `cli` has a `src/` directory — `agents-api` code is directly in `agents_api/`.
- **Large AI refactors in a single commit**: makes `git bisect` difficult; keep commits granular.
