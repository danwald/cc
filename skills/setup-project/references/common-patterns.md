# Common Patterns

Cross-language conventions that apply to all projects.

## Docker

**Always develop with Docker.** Every service — backend, frontend, database, cache, queue — runs in a container. Never run infrastructure services directly on the host.

### docker-compose.yml structure

```yaml
services:
  backend:
    build: { context: ./backend, dockerfile: Dockerfile }
    ports: ["8000:8000"]
    volumes: ["./backend:/app"]          # hot-reload in dev
    env_file: [./backend/.env]
    depends_on:
      postgres: { condition: service_healthy }

  frontend:
    build: { context: ./frontend, dockerfile: Dockerfile }
    ports: ["3000:3000"]
    volumes: ["./frontend:/app", "/app/node_modules"]
    environment:
      NEXT_PUBLIC_API_URL: http://localhost:8000
    depends_on: [backend]

  postgres:
    image: postgres:17-alpine
    ports: ["5432:5432"]
    environment: { POSTGRES_USER: app, POSTGRES_PASSWORD: app, POSTGRES_DB: app_dev }
    volumes: [postgres_data:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      retries: 5

volumes:
  postgres_data:
```

### Rules

| Rule | Detail |
|------|--------|
| **All services in Docker** | Backend, frontend, DB, cache — always containerised locally |
| **`depends_on` + healthcheck** | Don't start the app before Postgres is ready |
| **`volumes` for hot-reload** | Mount source into the container so changes reflect instantly |
| **Never `docker run` manually** | All orchestration goes through `docker-compose` |
| **Separate dev/prod Dockerfiles** | `Dockerfile` = prod multi-stage; `Dockerfile.dev` = dev with hot-reload if needed |

### Dockerfiles

**`backend/Dockerfile`** (Python / FastAPI):

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY pyproject.toml .
RUN pip install uv && uv pip install --system -e ".[dev]"
COPY . .
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

For hot-reload in dev, override the `command` in docker-compose:
`command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload`

**`frontend/Dockerfile`** (Node.js / Next.js):

```dockerfile
FROM node:20-slim
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["npm", "run", "dev"]
```

### Makefile (always include at project root)

```makefile
.PHONY: help build up down restart logs test lint clean

help: ## Show this help message
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | \
	  awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

# ── Docker ────────────────────────────────────────────────────────────────────
build: ## Build all Docker containers
	docker compose build
build-backend: ## Build backend only
	docker compose build backend
build-frontend: ## Build frontend only
	docker compose build frontend
up: ## Start all services (detached)
	docker compose up -d
up-attached: ## Start all services with logs
	docker compose up
down: ## Stop all services
	docker compose down
restart: ## Restart all services
	docker compose down && docker compose up -d
logs: ## Tail logs from all services
	docker compose logs -f
logs-backend: ## Tail backend logs
	docker compose logs -f backend
logs-frontend: ## Tail frontend logs
	docker compose logs -f frontend

# ── Shells ────────────────────────────────────────────────────────────────────
backend-shell: ## Shell into backend container
	docker compose exec backend bash
frontend-shell: ## Shell into frontend container
	docker compose exec frontend sh

# ── Backend (local) ──────────────────────────────────────────────────────────
backend-install: ## Install backend deps locally
	cd backend && uv pip install -e ".[dev]"
backend-run: ## Run backend locally
	cd backend && uvicorn app.main:app --reload
backend-lint: ## Lint + format backend
	cd backend && uv run ruff check --fix . && uv run ruff format .
backend-typecheck: ## Type-check backend
	cd backend && uv run mypy app/
test-backend: ## Run backend tests
	cd backend && uv run pytest -v

# ── Frontend (local) ─────────────────────────────────────────────────────────
frontend-install: ## Install frontend deps locally
	cd frontend && npm install
frontend-run: ## Run frontend locally
	cd frontend && npm run dev
frontend-lint: ## Lint frontend
	cd frontend && npm run lint
frontend-format: ## Format frontend
	cd frontend && npm run format
test-frontend: ## Run frontend unit tests
	cd frontend && npm run test:run

# ── Combined ─────────────────────────────────────────────────────────────────
test: test-backend test-frontend ## Run all tests
lint: backend-lint frontend-lint ## Lint all code

# ── Cleanup ───────────────────────────────────────────────────────────────────
clean: ## Remove containers, volumes, built images
	docker compose down -v --rmi local
```

---

## Architecture

### Hexagonal Architecture (Ports & Adapters)

All projects — backend and frontend — follow hexagonal architecture. The core rule: **dependencies point inward**. The domain knows nothing about frameworks, databases, or HTTP.

```
┌─────────────────────────────────────────────────┐
│  Infrastructure (adapters — outermost ring)     │
│  ┌─────────────────────────────────────────┐    │
│  │  Application (use cases / orchestration) │    │
│  │  ┌───────────────────────────────────┐  │    │
│  │  │  Domain (pure business logic)     │  │    │
│  │  │  No framework, DB, or HTTP deps   │  │    │
│  │  └───────────────────────────────────┘  │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
Dependency direction: Infrastructure → Application → Domain
```

#### Layer responsibilities

| Layer | Contains | Allowed imports |
|-------|----------|-----------------|
| **Domain** | Entities, value objects, domain services, port interfaces (ABCs / TS interfaces), domain exceptions | Nothing outside this layer |
| **Application** | Use cases, command/query handlers, application services | Domain only |
| **Infrastructure** | HTTP controllers, DB repositories, third-party adapters, DI wiring | Domain + Application |

#### Directory layout

```
# Python (FastAPI)
app/
├── domain/
│   ├── models/          # Entities, value objects
│   ├── repositories/    # Port interfaces (ABCs) — AIDEV-NOTE these
│   ├── services/        # Pure domain services
│   └── exceptions.py
├── application/
│   └── services/        # Use cases — orchestrate domain via injected ports
└── infrastructure/
    ├── api/             # FastAPI routes, DI wiring (dependencies.py)
    └── persistence/     # SQLAlchemy repository implementations

# TypeScript (Next.js)
src/
├── domain/
│   ├── entities/        # Pure TypeScript types / classes
│   ├── repositories/    # Repository interfaces (TypeScript interfaces)
│   └── services/        # Pure domain logic
├── application/
│   └── use-cases/       # Orchestrate domain via injected port interfaces
└── infrastructure/
    ├── db/              # Prisma repository implementations
    └── http/            # External API adapters
```

### Facades & Dependency Injection

**Facades are the mechanism for enforcing one-way dependencies.** Every cross-layer call goes through a port interface (Python ABC or TypeScript interface), never a concrete implementation.

#### Rules

1. **Domain defines the port** — the interface lives in `domain/repositories/` or `domain/services/`, never in infrastructure.
2. **Infrastructure implements the port** — the concrete class (e.g. `SqlAlchemyUserRepository`) lives in `infrastructure/` and satisfies the domain's interface.
3. **Application receives ports via constructor injection** — use cases never import concrete infrastructure classes.
4. **Wiring happens at the edge** — `infrastructure/api/dependencies.py` (FastAPI) or a DI container (TS) assembles concrete instances.

#### Python example

```python
# domain/repositories/__init__.py
# AIDEV-NOTE: port interface — infrastructure must implement, application depends on this
from abc import ABC, abstractmethod
from app.domain.models.user import User


class UserRepository(ABC):
    @abstractmethod
    async def get_by_id(self, user_id: int) -> User | None: ...

    @abstractmethod
    async def save(self, user: User) -> User: ...


# application/services/create_user.py
# AIDEV-NOTE: use case — depends only on the port interface, never on SQLAlchemy
from app.domain.repositories import UserRepository
from app.domain.models.user import User


class CreateUserService:
    def __init__(self, user_repo: UserRepository) -> None:
        self._user_repo = user_repo  # injected — could be real DB or test fake

    async def execute(self, name: str, email: str) -> User:
        user = User(name=name, email=email)
        return await self._user_repo.save(user)


# infrastructure/api/dependencies.py
# AIDEV-NOTE: DI wiring — only place that knows about concrete adapters
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from app.database import get_db
from app.domain.repositories import UserRepository
from app.infrastructure.persistence.repositories.user_repo import SqlAlchemyUserRepository
from app.application.services.create_user import CreateUserService


async def get_user_repository(db: AsyncSession = Depends(get_db)) -> UserRepository:
    return SqlAlchemyUserRepository(db)


async def get_create_user_service(
    user_repo: UserRepository = Depends(get_user_repository),
) -> CreateUserService:
    return CreateUserService(user_repo)
```

#### TypeScript example

```typescript
// domain/repositories/user-repository.ts
// AIDEV-NOTE: port interface — concrete implementation lives in infrastructure/
export interface UserRepository {
  findById(id: string): Promise<User | null>
  save(user: User): Promise<User>
}

// application/use-cases/create-user.ts
// AIDEV-NOTE: use case — depends only on the port interface, never on Prisma
import type { UserRepository } from "@/domain/repositories/user-repository"

export class CreateUserUseCase {
  constructor(private readonly userRepo: UserRepository) {}

  async execute(name: string, email: string): Promise<User> {
    const user = new User({ name, email })
    return this.userRepo.save(user)
  }
}

// infrastructure/db/prisma-user-repository.ts
// AIDEV-NOTE: concrete adapter — satisfies the domain port using Prisma
import { prisma } from "@/lib/prisma"
import type { UserRepository } from "@/domain/repositories/user-repository"

export class PrismaUserRepository implements UserRepository {
  async findById(id: string) { ... }
  async save(user: User) { ... }
}
```

### Anchor Comments for Architecture

Always add `AIDEV-NOTE:` comments at key architectural boundaries:

```python
# AIDEV-NOTE: port interface — infrastructure must implement, application depends on this
class UserRepository(ABC): ...

# AIDEV-NOTE: use case — depends only on port interfaces, never concrete adapters
class CreateUserService: ...

# AIDEV-NOTE: DI wiring — only place that knows about concrete adapter implementations
async def get_user_repository(...): ...

# AIDEV-NOTE: infrastructure adapter — implements domain port; keep framework deps here
class SqlAlchemyUserRepository(UserRepository): ...
```

These comments let any future AI or developer immediately understand which layer a file belongs to and what it's allowed to depend on.

---

## Git

### Universal .gitignore

```gitignore
# Python
__pycache__/
*.py[cod]
.venv/
venv/
*.egg-info/
dist/
*.db
.coverage
htmlcov/
coverage/

# Node
node_modules/
.next/
.env.local

# IDE
.vscode/
.idea/

# OS
.DS_Store
*.swp

# Secrets
.env

# Testing
.coverage
htmlcov/
```

### Universal .editorconfig

```ini
root = true

[*]
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
charset = utf-8

[*.md]
trim_trailing_whitespace = false
```

## README Essentials

Every project README needs:
1. One-line description
2. Quick start (install + test)
3. Prerequisites
4. Commands table
5. Project structure

## Pre-commit Order

Run tools fastest-first for quick feedback:

1. **Format** (fast, auto-fixable)
2. **Lint** (fast, static analysis)
3. **Type check** (medium speed)
4. **Tests** (slow — run on changed files only via lint-staged/pre-commit)

Python: use `pre-commit`. JS/TS: use `lint-staged` with `husky`.

## CI Order

Same as pre-commit: **format → lint → typecheck → test**. Fast gates fail early and cheaply.

## Testing Philosophy

| Type | What | When |
|------|------|------|
| Unit | Pure functions, business logic | Always |
| Integration | Database, API endpoints | Critical paths |
| E2E | User journeys | Happy paths only |

**Mocking rules:**
- Mock at boundaries: HTTP, filesystem, time, random, third-party APIs.
- Don't mock: your own code, database in integration tests.

**Structure:** Arrange → Act → Assert.

**Coverage gate:** 80% minimum (statements, functions, lines); 75% branches.

## Environment Variables

```
.env           # Defaults (committed, no secrets)
.env.local     # Local overrides (gitignored)
```

Never commit secrets. Rotate immediately if exposed. Use `os.environ["KEY"]` (Python) or `process.env.KEY` (JS/TS). Store CI secrets in GitHub Secrets.

## Versioning

Semantic: `MAJOR.MINOR.PATCH` (breaking.feature.fix), as specified in each component's `pyproject.toml`.

| Bump | When |
|------|------|
| **MAJOR** | Incompatible API changes |
| **MINOR** | Backward-compatible new functionality |
| **PATCH** | Backward-compatible bug fixes |

## Dependencies

Update order: dev deps → patch → minor → major (test at each step).

Always commit lock files (`uv.lock`, `package-lock.json`, `pnpm-lock.yaml`) for reproducible builds.

## Security

- Enable Dependabot or Renovate for automated dependency updates.
- Run security audits in CI: `uvx pip-audit` (Python), `npm audit` (JS/TS).
- Never commit secrets — use `.env.local` or CI secrets.
- Rotate credentials immediately if exposed.

## Commit Discipline

- **Granular commits**: one logical change per commit.
- **Tag AI-generated commits**: e.g. `feat: optimise feed query [AI]`.
- **Clear messages**: explain the *why*; link to issues/ADRs for architectural changes.
- **Use `git worktree`** for parallel/long-running AI branches (e.g. `git worktree add ../wip-foo -b wip-foo`).
- **Review AI-generated code**: never merge code you don't understand.

## Documentation

- **README**: setup, usage, contribution guide.
- **Code comments**: explain *why*, not *what*.
- **Types**: document the shape of data and constraints.
- **Tests**: document expected behavior.
- **Anchor comments**: use `AIDEV-NOTE:`, `AIDEV-TODO:`, `AIDEV-QUESTION:` for inline knowledge (≤120 chars).
