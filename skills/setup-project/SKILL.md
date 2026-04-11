---
name: setup-project
description: Guide for setting up Python or TypeScript/frontend projects following project conventions. Use when the user asks to set up a new project, asks about Python/TypeScript coding standards, needs help configuring tools (uv, ruff, mypy, pytest, ESLint, Prettier, Vitest), configuring pre-commit hooks or CI pipelines, or wants to understand project conventions and best practices.
user-invocable: false
---

# Project Setup Guide

Authoritative standards for Python and TypeScript/frontend project setup.

## References

- [Python standards](references/python.md) — uv, ruff, mypy, pytest, coding conventions, Ward testing, error handling, agents-api expressions
- [TypeScript / Frontend standards](references/typescript.md) — Node.js, TypeScript, React/Next.js, ESLint, Prettier, Vitest, Playwright
- [Datasource setup](references/datasource.md) — PostgreSQL, Docker-based local Postgres, SQLAlchemy/asyncpg (Python), Prisma (TS), Alembic/Prisma migrations, integration testing against real DB
- [Common patterns](references/common-patterns.md) — Docker, hexagonal architecture, facades & DI, git, CI/CD, testing philosophy, environment variables, versioning, security

## When to Use

Load the relevant reference when:
- Starting a new Python or TypeScript/frontend project
- Asked about coding standards, tool configuration, or best practices
- Setting up linting, formatting, type checking, or testing
- Configuring pre-commit hooks or CI pipelines
- Reviewing or updating `pyproject.toml`, `tsconfig.json`, or CI workflows
- Setting up a database, migrations, or Docker services
- Designing the layer/dependency structure of a new feature or service

## Quick Reference

### Python
```bash
uv run ruff check --fix .    # Lint + auto-fix
uv run ruff format .         # Format
uv run mypy                  # Type check
uv run pytest                # Run all tests
uv run pytest -k "pattern"   # Run specific tests
```

### TypeScript
```bash
npm run lint                 # ESLint
npm run format               # Prettier
npx tsc --noEmit             # Type check
npm run test                 # Vitest
npm run test:run             # Vitest (CI, no watch)
```
