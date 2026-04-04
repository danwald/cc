---
name: fullstack-setup
description: Scaffold a fullstack application with a FastAPI backend (DDD hexagonal architecture) and a Next.js (TypeScript/React) frontend. Use when the user wants to create a new fullstack project.
disable-model-invocation: true
argument-hint: [project-name]
allowed-tools: Bash, Write, Read, Glob, Grep, Edit
---

# Fullstack App Setup: FastAPI + Next.js (TypeScript/React)

Set up a fullstack application named `$ARGUMENTS` (default to `fullstack-app` if no name provided).

- **Backend**: FastAPI with DDD Hexagonal Architecture (Ports & Adapters), SQLite (dev) with FK support
- **Frontend**: Next.js App Router, TypeScript (strict), React 19, Tailwind CSS v4

---

## Backend Best Practices

Follow these conventions strictly for all generated Python code:

### Coding Standards
- **Python 3.12+**, FastAPI, `async/await` preferred
- **Typing**: Strict — Pydantic v2 models preferred; use `from __future__ import annotations` in every file
- **Formatting**: `ruff` enforces 96-char lines, double quotes, sorted imports
- **Naming**: `snake_case` (functions/variables), `PascalCase` (classes), `SCREAMING_SNAKE` (constants)
- **Docstrings**: Google-style for public functions/classes
- **Error Handling**: Typed, hierarchical exceptions in `exceptions.py`; catch specific exceptions, not generic `Exception`; use context managers for resources (DB connections, file handles); use `try/finally` in async code for cleanup
- **Anchor Comments**: Add `AIDEV-NOTE:` comments near non-trivial code (keep ≤120 chars). Use `AIDEV-TODO:` for planned work and `AIDEV-QUESTION:` for unclear areas
- **Package Manager**: Use `uv` for running applications with project context (`uv run ruff`, `uv run mypy`, `uv run pytest`)

### Architecture Rules (Hexagonal DDD)

The domain layer is pure Python with no framework dependencies. The application layer orchestrates use cases. Infrastructure adapters (database, HTTP) live at the edges.

1. **Domain layer** (`domain/`) has ZERO imports from `application/`, `infrastructure/`, FastAPI, or SQLAlchemy (except the declarative base in `models/base.py`)
2. **Application layer** (`application/`) may import from `domain/` only. It receives infrastructure via dependency injection (constructor arguments typed to domain port interfaces)
3. **Infrastructure layer** (`infrastructure/`) may import from both `domain/` and `application/`. It implements the port interfaces and wires everything together
4. **Dependency direction**: `infrastructure/ → application/ → domain/` (never the reverse)

---

## Frontend Best Practices

Follow these conventions strictly for all generated TypeScript/React code:

### TypeScript
- **Strict mode**: All strict flags enabled in `tsconfig.json`, plus `noUnusedLocals`, `noUnusedParameters`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`
- **Prefer `type` over `interface`** for props, unions, intersections. Use `interface` only when you need declaration merging or `implements`
- **No `any`**: Use `unknown` and narrow with type guards. Use `Record<string, unknown>` for generic objects
- **Export prop types** alongside their components

### React Patterns
- **Default to Server Components.** Only add `"use client"` when you need state, effects, event handlers, or browser APIs
- **Push `"use client"` as deep in the tree as possible** — extract small interactive pieces into focused Client Components
- **One component per file** — kebab-case filenames (`user-profile.tsx`), PascalCase exports (`UserProfile`)
- **Co-locate** tests, styles, and sub-components next to the component they belong to
- **React Compiler**: When available (Next.js 16+), `useMemo`/`useCallback`/`React.memo` are automatic. Don't add them manually

### Next.js App Router
- Use **route groups** `(marketing)/`, `(app)/` for separate layouts without affecting URL paths
- Use **private folders** `_components/` for co-located route helpers excluded from routing
- Use special files: `layout.tsx`, `loading.tsx`, `error.tsx` (must be `"use client"`), `not-found.tsx`, `global-error.tsx`
- **Server Actions**: Always validate with Zod, re-verify auth inside every action, return errors as values (don't throw)
- **Metadata API**: Use static `metadata` export or `generateMetadata()` for dynamic SEO

### Styling (Tailwind CSS v4)
- CSS-first config via `@theme` directive (no `tailwind.config.js`)
- Create a `cn()` utility using `clsx` + `tailwind-merge` in `src/lib/utils.ts`
- Use `prettier-plugin-tailwindcss` to auto-sort class names
- Avoid `@apply` — write utility classes directly or extract components

### State Management
- **Server state**: Fetch directly in Server Components; use TanStack Query in Client Components for cache/polling/optimistic updates
- **Global client state**: Zustand (if needed)
- **Form state**: React Hook Form + Zod (or `useActionState` with Server Actions)
- **URL state**: `nuqs` for search params as state
- **React Context**: Only for low-frequency updates (theme, locale, auth session). Never for frequently updating state

### Performance
- `next/image` with `priority` for LCP images, always provide `sizes` for responsive images
- `next/font/google` with `variable` option for self-hosted fonts with zero layout shift
- `dynamic()` imports with `ssr: false` for browser-only heavy components

### Testing
- **Unit/Component**: Vitest + React Testing Library (co-locate as `*.test.tsx`)
- **E2E**: Playwright (`tests/e2e/*.spec.ts`)
- **API mocking**: MSW (Mock Service Worker)

### Linting & Formatting
- ESLint flat config (`eslint.config.mjs`) with `next/core-web-vitals`, `next/typescript`, `@typescript-eslint/strict-type-checked`, `jsx-a11y/recommended`, and `prettier` (last)
- Prettier with `prettier-plugin-tailwindcss`
- `husky` + `lint-staged` for pre-commit hooks

### Security
- Use `server-only` import on modules that must never leak to the client
- Server Actions auto-verify `Origin` vs `Host` (CSRF). Custom Route Handlers need manual CSRF protection
- Data Access Layer: Always verify auth before returning data, return only safe fields
- Security headers in `next.config.ts`: `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Strict-Transport-Security`

### Error Handling
- `error.tsx` per route segment (must be `"use client"`) — catches errors in `page.tsx` and children, NOT the same-segment `layout.tsx`
- `global-error.tsx` at app root (must render its own `<html>` and `<body>`)
- Server Actions: Return errors as values with Zod validation, don't throw
- API routes: `try/catch` with structured error responses

---

## Project Structure

```
$ARGUMENTS/
├── backend/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── database.py
│   │   │
│   │   ├── domain/                       # Inner core — pure business logic
│   │   │   ├── __init__.py
│   │   │   ├── models/
│   │   │   │   ├── __init__.py
│   │   │   │   └── base.py              # SQLAlchemy declarative base
│   │   │   ├── repositories/            # Port interfaces (ABCs)
│   │   │   │   └── __init__.py
│   │   │   ├── services/                # Domain services (pure rules)
│   │   │   │   └── __init__.py
│   │   │   └── exceptions.py
│   │   │
│   │   ├── application/                  # Use cases — orchestrates domain
│   │   │   ├── __init__.py
│   │   │   └── services/
│   │   │       └── __init__.py
│   │   │
│   │   └── infrastructure/               # Outer ring — adapters
│   │       ├── __init__.py
│   │       ├── api/                      # Primary adapter (HTTP)
│   │       │   ├── __init__.py
│   │       │   ├── dependencies.py       # FastAPI DI wiring
│   │       │   └── routes/
│   │       │       ├── __init__.py
│   │       │       └── health.py
│   │       └── persistence/              # Secondary adapter (DB)
│   │           ├── __init__.py
│   │           └── repositories/
│   │               └── __init__.py
│   │
│   ├── tests/
│   │   ├── __init__.py
│   │   ├── conftest.py
│   │   ├── test_health.py
│   │   ├── domain/
│   │   │   └── __init__.py
│   │   ├── application/
│   │   │   └── __init__.py
│   │   └── infrastructure/
│   │       └── __init__.py
│   │
│   ├── pyproject.toml
│   ├── Dockerfile
│   └── .env.example
│
├── frontend/
│   ├── src/
│   │   ├── app/                          # Next.js App Router (routing only)
│   │   │   ├── layout.tsx                # Root layout (fonts, providers)
│   │   │   ├── page.tsx                  # Landing page
│   │   │   ├── loading.tsx               # Root loading skeleton
│   │   │   ├── error.tsx                 # Root error boundary ("use client")
│   │   │   ├── not-found.tsx             # 404 page
│   │   │   ├── global-error.tsx          # Catches root layout errors
│   │   │   └── api/
│   │   │       └── health/route.ts       # Example API route (optional proxy)
│   │   ├── components/
│   │   │   ├── ui/                       # Reusable primitives (Button, Input)
│   │   │   ├── layout/                   # Shell (Header, Footer, Sidebar)
│   │   │   └── features/                 # Domain-specific composites
│   │   ├── hooks/                        # Custom React hooks
│   │   ├── lib/
│   │   │   ├── utils.ts                  # cn() and other helpers
│   │   │   └── api.ts                    # API client
│   │   ├── types/                        # Shared TypeScript types
│   │   │   └── index.ts
│   │   └── styles/
│   │       └── globals.css
│   ├── tests/
│   │   └── e2e/                          # Playwright E2E tests
│   ├── public/
│   ├── vitest.config.mts
│   ├── eslint.config.mjs
│   ├── .prettierrc
│   ├── Dockerfile
│   ├── .env.local
│   └── .env.example
│
├── docker-compose.yml
├── Makefile
├── .gitignore
└── README.md
```

---

## Step-by-step Instructions

### 1. Create root project directory

Create the root directory and initialize a git repository.

### 2. Backend Setup

#### `backend/pyproject.toml`

Use `pyproject.toml` (not `requirements.txt`) with `uv` as the package manager:

```toml
[project]
name = "$ARGUMENTS-backend"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.34.0",
    "pydantic>=2.0",
    "pydantic-settings>=2.0",
    "python-dotenv>=1.0",
    "sqlalchemy[asyncio]>=2.0",
    "aiosqlite>=0.20.0",
]

[project.optional-dependencies]
dev = [
    "httpx>=0.28.0",
    "pytest>=8.0",
    "pytest-asyncio>=0.25.0",
    "ruff>=0.8.0",
    "mypy>=1.13.0",
]

[tool.ruff]
line-length = 96
target-version = "py312"

[tool.ruff.format]
quote-style = "double"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM", "ASYNC"]

[tool.ruff.lint.isort]
known-first-party = ["app"]

[tool.pytest.ini_options]
asyncio_mode = "auto"

[tool.mypy]
python_version = "3.12"
strict = true
```

#### `backend/.env.example`

```
APP_ENV=development
APP_DEBUG=true
CORS_ORIGINS=http://localhost:3000
DATABASE_URL=sqlite+aiosqlite:///./app.db
```

#### `backend/app/config.py`

Use pydantic-settings `BaseSettings` to load env vars:
- `APP_ENV`, `APP_DEBUG`, `CORS_ORIGINS` (as a comma-separated list), `DATABASE_URL`
- Include `from __future__ import annotations`

---

#### Domain Layer

**`backend/app/domain/models/base.py`** — SQLAlchemy declarative base:

```python
from __future__ import annotations

from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass
```

Re-export `Base` from `backend/app/domain/models/__init__.py`.

**`backend/app/domain/exceptions.py`** — Typed, hierarchical domain exceptions:

```python
from __future__ import annotations


class DomainException(Exception):
    """Base exception for all domain errors."""


class EntityNotFoundError(DomainException):
    """Raised when a requested entity does not exist."""


class ValidationError(DomainException):
    """Raised when domain validation fails."""
```

**`backend/app/domain/repositories/__init__.py`** — Port interfaces with sample abstract repository:

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from typing import TypeVar

T = TypeVar("T")


class BaseRepository(ABC):
    """Port interface — implemented by infrastructure adapters."""

    @abstractmethod
    async def get_by_id(self, entity_id: int) -> T | None:
        ...

    @abstractmethod
    async def save(self, entity: T) -> T:
        ...
```

---

#### Application Layer

**`backend/app/application/services/__init__.py`** — Placeholder with docstring:

```python
"""Application services (use cases).

Each use case receives repository ports via constructor injection.
Example:

    class CreateUserService:
        def __init__(self, user_repo: UserRepository) -> None:
            self._user_repo = user_repo

        async def execute(self, data: CreateUserInput) -> User:
            ...
"""
```

---

#### Infrastructure Layer

**`backend/app/database.py`** — Database engine with SQLite foreign key support:

```python
from __future__ import annotations

from sqlalchemy import event
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from app.config import settings

engine = create_async_engine(settings.DATABASE_URL, echo=settings.APP_DEBUG)
async_session = async_sessionmaker(engine, expire_on_commit=False)


# AIDEV-NOTE: Enable SQLite FK support on every raw connection checkout
@event.listens_for(engine.sync_engine, "connect")
def _set_sqlite_pragma(dbapi_connection, connection_record):
    cursor = dbapi_connection.cursor()
    cursor.execute("PRAGMA foreign_keys=ON")
    cursor.close()


async def get_db() -> AsyncSession:
    """FastAPI dependency that yields a database session."""
    async with async_session() as session:
        yield session
```

**`backend/app/infrastructure/api/dependencies.py`** — FastAPI DI wiring:

```python
from __future__ import annotations

# AIDEV-NOTE: Bind port interfaces to concrete adapters here
# Example:
# from fastapi import Depends
# from app.domain.repositories import UserRepository
# from app.infrastructure.persistence.repositories.user_repo import SqlAlchemyUserRepository
# from app.database import get_db
#
# async def get_user_repository(db=Depends(get_db)) -> UserRepository:
#     return SqlAlchemyUserRepository(db)
```

**`backend/app/infrastructure/api/routes/health.py`** — Health check:
- `GET /api/health` returning `{"status": "healthy"}`

Wire up the router in `backend/app/infrastructure/api/__init__.py` with prefix `/api`.

**`backend/app/main.py`** — FastAPI app with lifespan:

```python
from __future__ import annotations

from contextlib import asynccontextmanager

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.config import settings
from app.database import engine
from app.domain.models.base import Base
from app.infrastructure.api import api_router


@asynccontextmanager
async def lifespan(app: FastAPI):
    # AIDEV-NOTE: Creates all tables on startup (dev only; use migrations in prod)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield


app = FastAPI(title="$ARGUMENTS API", lifespan=lifespan)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins_list,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(api_router)


@app.get("/")
async def root():
    return {"message": "API is running"}
```

---

#### Tests

**`backend/tests/conftest.py`** — Shared fixtures with SQLite FK support:

```python
from __future__ import annotations

import pytest
from sqlalchemy import event
from sqlalchemy.ext.asyncio import async_sessionmaker, create_async_engine

from app.database import get_db
from app.domain.models.base import Base
from app.main import app

TEST_DATABASE_URL = "sqlite+aiosqlite:///:memory:"


@pytest.fixture
async def db_session():
    test_engine = create_async_engine(TEST_DATABASE_URL)

    @event.listens_for(test_engine.sync_engine, "connect")
    def _set_sqlite_pragma(dbapi_connection, connection_record):
        cursor = dbapi_connection.cursor()
        cursor.execute("PRAGMA foreign_keys=ON")
        cursor.close()

    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    session_factory = async_sessionmaker(test_engine, expire_on_commit=False)
    async with session_factory() as session:
        app.dependency_overrides[get_db] = lambda: session
        yield session
        app.dependency_overrides.clear()

    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await test_engine.dispose()
```

**`backend/tests/test_health.py`** — Test using `httpx.AsyncClient` and `pytest-asyncio`:
- Test that `/api/health` returns 200 and `{"status": "healthy"}`

---

### 3. Frontend Setup

#### Scaffold with create-next-app

```bash
npx create-next-app@latest frontend --typescript --tailwind --eslint --app --src-dir --import-alias "@/*" --use-npm
```

#### After scaffolding, add/modify these files:

**`frontend/src/lib/utils.ts`** — cn() utility:

```typescript
import { type ClassValue, clsx } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

**`frontend/src/lib/api.ts`** — Type-safe API client:

```typescript
const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL || "http://localhost:8000"

export async function fetchApi<T>(endpoint: string, options?: RequestInit): Promise<T> {
  const res = await fetch(`${API_BASE_URL}${endpoint}`, {
    headers: { "Content-Type": "application/json", ...options?.headers },
    ...options,
  })
  if (!res.ok) {
    throw new Error(`API error: ${res.status}`)
  }
  return res.json() as Promise<T>
}
```

**`frontend/src/app/error.tsx`** — Root error boundary:

```tsx
"use client"

import { useEffect } from "react"

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  useEffect(() => {
    console.error(error)
  }, [error])

  return (
    <div className="flex min-h-screen flex-col items-center justify-center gap-4">
      <h2 className="text-xl font-semibold">Something went wrong</h2>
      <button
        onClick={() => reset()}
        className="rounded-md bg-blue-600 px-4 py-2 text-white hover:bg-blue-700"
      >
        Try again
      </button>
    </div>
  )
}
```

**`frontend/src/app/not-found.tsx`** — 404 page

**`frontend/src/app/loading.tsx`** — Root loading skeleton

**`frontend/src/app/global-error.tsx`** — Catches root layout errors (must render `<html>` and `<body>`, must be `"use client"`)

Update **`frontend/src/app/layout.tsx`** to use `next/font/google` with CSS variables:

```tsx
import type { Metadata } from "next"
import { Geist, Geist_Mono } from "next/font/google"
import "@/styles/globals.css"

const geist = Geist({ subsets: ["latin"], variable: "--font-sans" })
const geistMono = Geist_Mono({ subsets: ["latin"], variable: "--font-mono" })

export const metadata: Metadata = {
  title: "$ARGUMENTS",
  description: "Fullstack application",
}

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={`${geist.variable} ${geistMono.variable}`}>
      <body className="font-sans antialiased">{children}</body>
    </html>
  )
}
```

Update **`frontend/src/app/page.tsx`** to show a landing page that fetches and displays the backend health status on mount, demonstrating the frontend-to-backend connection works. Use a small `"use client"` component for the health check fetch, keeping the page itself as a Server Component.

**`frontend/src/types/index.ts`** — Shared types placeholder

**`frontend/.env.local`** and **`frontend/.env.example`**:

```
NEXT_PUBLIC_API_URL=http://localhost:8000
```

#### Install additional dependencies

```bash
cd frontend && npm install clsx tailwind-merge zod
```

#### Configure Vitest

```bash
cd frontend && npm install -D vitest @vitejs/plugin-react @testing-library/react @testing-library/dom @testing-library/user-event vite-tsconfig-paths jsdom
```

**`frontend/vitest.config.mts`**:

```typescript
import react from "@vitejs/plugin-react"
import tsconfigPaths from "vite-tsconfig-paths"
import { defineConfig } from "vitest/config"

export default defineConfig({
  plugins: [tsconfigPaths(), react()],
  test: {
    environment: "jsdom",
    globals: true,
    setupFiles: ["./src/test/setup.ts"],
    include: ["src/**/*.test.{ts,tsx}"],
  },
})
```

**`frontend/src/test/setup.ts`**:

```typescript
import "@testing-library/jest-dom/vitest"
```

Install: `npm install -D @testing-library/jest-dom`

#### Configure Prettier

**`frontend/.prettierrc`**:

```json
{
  "semi": false,
  "singleQuote": false,
  "tabWidth": 2,
  "trailingComma": "all",
  "printWidth": 100,
  "plugins": ["prettier-plugin-tailwindcss"]
}
```

Install: `npm install -D prettier prettier-plugin-tailwindcss`

#### Update tsconfig.json

Ensure these strict flags are set:

```jsonc
{
  "compilerOptions": {
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noUncheckedIndexedAccess": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

#### Configure Next.js security headers

In **`frontend/next.config.ts`**, add security headers:

```typescript
const nextConfig = {
  async headers() {
    return [{
      source: "/(.*)",
      headers: [
        { key: "X-Frame-Options", value: "DENY" },
        { key: "X-Content-Type-Options", value: "nosniff" },
        { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
        { key: "Permissions-Policy", value: "camera=(), microphone=(), geolocation=()" },
      ],
    }]
  },
}
```

#### Add test scripts to package.json

```json
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run",
    "test:coverage": "vitest run --coverage",
    "lint": "next lint",
    "format": "prettier --write \"src/**/*.{ts,tsx,css}\""
  }
}
```

---

### 4. Docker Compose (development)

**`docker-compose.yml`** at project root:

```yaml
services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    volumes:
      - ./backend:/app
    env_file:
      - ./backend/.env.example
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    volumes:
      - ./frontend:/app
      - /app/node_modules
    environment:
      - NEXT_PUBLIC_API_URL=http://localhost:8000
    command: npm run dev
    depends_on:
      - backend
```

**`backend/Dockerfile`**:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY pyproject.toml .
RUN pip install uv && uv pip install --system -e ".[dev]"
COPY . .
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**`frontend/Dockerfile`**:

```dockerfile
FROM node:20-slim
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["npm", "run", "dev"]
```

---

### 5. Makefile

Create a `Makefile` at the project root:

```makefile
.PHONY: help build up down restart logs test lint clean

help: ## Show this help message
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

# ── Docker ────────────────────────────────────────────────────
build: ## Build all Docker containers
	docker compose build

build-backend: ## Build only the backend container
	docker compose build backend

build-frontend: ## Build only the frontend container
	docker compose build frontend

up: ## Start all services in the background
	docker compose up -d

up-attached: ## Start all services with logs attached
	docker compose up

down: ## Stop all services
	docker compose down

restart: ## Restart all services
	docker compose down && docker compose up -d

logs: ## Tail logs from all services
	docker compose logs -f

logs-backend: ## Tail logs from backend only
	docker compose logs -f backend

logs-frontend: ## Tail logs from frontend only
	docker compose logs -f frontend

# ── Shells ────────────────────────────────────────────────────
backend-shell: ## Open a shell in the backend container
	docker compose exec backend bash

frontend-shell: ## Open a shell in the frontend container
	docker compose exec frontend sh

# ── Backend (local) ──────────────────────────────────────────
backend-install: ## Install backend dependencies locally with uv
	cd backend && uv pip install -e ".[dev]"

backend-run: ## Run backend locally
	cd backend && uvicorn app.main:app --reload

backend-lint: ## Lint and format backend code
	cd backend && uv run ruff check --fix . && uv run ruff format .

backend-typecheck: ## Run mypy type checking
	cd backend && uv run mypy app/

test-backend: ## Run backend tests
	cd backend && uv run pytest -v

# ── Frontend (local) ─────────────────────────────────────────
frontend-install: ## Install frontend dependencies locally
	cd frontend && npm install

frontend-run: ## Run frontend locally
	cd frontend && npm run dev

frontend-lint: ## Lint frontend code
	cd frontend && npm run lint

frontend-format: ## Format frontend code
	cd frontend && npm run format

test-frontend: ## Run frontend unit tests
	cd frontend && npm run test:run

# ── Full stack ───────────────────────────────────────────────
test: test-backend test-frontend ## Run all tests

lint: backend-lint frontend-lint ## Lint all code

# ── Cleanup ──────────────────────────────────────────────────
clean: ## Remove all containers, volumes, and built images
	docker compose down -v --rmi local
```

---

### 6. Root Files

**`.gitignore`**:

```
# Python
__pycache__/
*.py[cod]
.env
.venv/
venv/
*.db
*.egg-info/
dist/

# Node
node_modules/
.next/
.env.local

# IDE
.vscode/
.idea/

# OS
.DS_Store

# Testing
coverage/
.coverage
htmlcov/
```

---

### 7. Post-Setup

After creating all files:
1. Copy `backend/.env.example` to `backend/.env`
2. Print a summary of what was created
3. Print instructions:

```
## Getting Started

### Using Make (recommended)
make build        # Build Docker containers
make up           # Start all services
make logs         # Tail logs
make test         # Run all tests
make lint         # Lint all code
make down         # Stop all services
make help         # See all available commands

### Backend (local, without Docker)
cd $ARGUMENTS/backend
uv venv && source .venv/bin/activate
uv pip install -e ".[dev]"
cp .env.example .env
uvicorn app.main:app --reload

### Frontend (local, without Docker)
cd $ARGUMENTS/frontend
npm install
npm run dev

Backend:  http://localhost:8000
Frontend: http://localhost:3000
API Docs: http://localhost:8000/docs

## Architecture

### Backend (Hexagonal DDD)
backend/app/
├── domain/          # Pure business logic — no framework imports
│   ├── models/      # Entities & value objects
│   ├── repositories/# Port interfaces (ABCs)
│   ├── services/    # Domain rules
│   └── exceptions.py
├── application/     # Use cases — orchestrates domain via ports
│   └── services/
└── infrastructure/  # Adapters — implements ports
    ├── api/         # HTTP (FastAPI routes, DI wiring)
    └── persistence/ # Database (SQLAlchemy repositories)

Dependency flow: infrastructure → application → domain

### Frontend
frontend/src/
├── app/             # Next.js routing (pages, layouts, error boundaries)
├── components/      # UI primitives, layout shells, feature components
│   ├── ui/          # Button, Input, Card — reusable primitives
│   ├── layout/      # Header, Footer, Sidebar
│   └── features/    # Domain-specific composites
├── hooks/           # Custom React hooks
├── lib/             # Utilities (cn, api client)
├── types/           # Shared TypeScript types
└── styles/          # Global CSS
```
