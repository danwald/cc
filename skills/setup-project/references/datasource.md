# Datasource Setup

PostgreSQL is the database of choice for all projects. Always run Postgres via Docker for local development — never install it directly on the host.

## Docker Compose (local dev)

Add a `postgres` service to `docker-compose.yml`:

```yaml
services:
  postgres:
    image: postgres:17-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: app_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  postgres_data:
```

**Connection string**: `postgresql+asyncpg://app:app@localhost:5432/app_dev` (Python async) or `postgresql://app:app@localhost:5432/app_dev` (sync / Prisma).

## Environment Variables

```bash
# .env.example  (committed — no secrets)
DATABASE_URL=postgresql+asyncpg://app:app@localhost:5432/app_dev

# .env.local (gitignored — local overrides)
DATABASE_URL=postgresql+asyncpg://app:app@localhost:5432/app_dev
```

For testing, use a separate database:

```bash
TEST_DATABASE_URL=postgresql+asyncpg://app:app@localhost:5432/app_test
```

---

## Python (SQLAlchemy async + asyncpg)

### Dependencies

```toml
[project]
dependencies = [
    "sqlalchemy[asyncio]>=2.0",
    "asyncpg>=0.30",
    "alembic>=1.14",
]
```

### Engine Setup

```python
# AIDEV-NOTE: engine is the infrastructure adapter for the database port
from __future__ import annotations

from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from app.config import settings

engine = create_async_engine(
    settings.DATABASE_URL,
    echo=settings.APP_DEBUG,
    pool_size=10,
    max_overflow=20,
)
async_session = async_sessionmaker(engine, expire_on_commit=False)


async def get_db() -> AsyncSession:
    """FastAPI dependency yielding a database session."""
    async with async_session() as session:
        yield session
```

### Migrations (Alembic)

```bash
# Initialise (once per project)
alembic init -t async alembic

# Generate a migration from model changes
alembic revision --autogenerate -m "add users table"

# Apply migrations
alembic upgrade head

# Roll back one step
alembic downgrade -1
```

`alembic/env.py` must import your `Base.metadata` and use an async connection:

```python
from app.domain.models.base import Base
target_metadata = Base.metadata
```

### Testing Against Real Postgres

Integration tests must hit a real Postgres — not SQLite, not mocks. Use a dedicated test database spun up via Docker:

```python
# conftest.py
import pytest
from sqlalchemy.ext.asyncio import async_sessionmaker, create_async_engine

from app.config import settings
from app.domain.models.base import Base

TEST_DATABASE_URL = settings.TEST_DATABASE_URL  # separate DB, same Postgres container


@pytest.fixture(scope="session")
async def engine():
    engine = create_async_engine(TEST_DATABASE_URL)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()


@pytest.fixture
async def db_session(engine):
    async with async_sessionmaker(engine, expire_on_commit=False)() as session:
        yield session
        await session.rollback()  # isolate each test
```

---

## TypeScript (Prisma)

### Dependencies

```bash
npm install @prisma/client
npm install -D prisma
```

### Setup

```bash
npx prisma init --datasource-provider postgresql
```

`prisma/schema.prisma`:

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}
```

### Prisma Client (singleton)

```typescript
// src/lib/prisma.ts
// AIDEV-NOTE: singleton Prisma client — infrastructure adapter for the DB port
import { PrismaClient } from "@prisma/client"

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient }

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({ log: process.env.NODE_ENV === "development" ? ["query"] : [] })

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma
```

### Migrations

```bash
# Create and apply a migration
npx prisma migrate dev --name add_users_table

# Apply in CI/production (no schema drift check)
npx prisma migrate deploy

# Regenerate the client after schema changes
npx prisma generate
```

### Testing Against Real Postgres

```typescript
// vitest.setup.ts
import { execSync } from "node:child_process"
import { prisma } from "@/lib/prisma"

beforeAll(() => {
  execSync("npx prisma migrate deploy", { env: { ...process.env, DATABASE_URL: process.env.TEST_DATABASE_URL } })
})

afterEach(async () => {
  // Truncate all tables between tests for isolation
  const tables = await prisma.$queryRaw<{ tablename: string }[]>`
    SELECT tablename FROM pg_tables WHERE schemaname = 'public'
  `
  for (const { tablename } of tables) {
    await prisma.$executeRawUnsafe(`TRUNCATE TABLE "${tablename}" RESTART IDENTITY CASCADE`)
  }
})

afterAll(async () => {
  await prisma.$disconnect()
})
```

### FastAPI dependency override in tests

Override FastAPI's `get_db` dependency so tests use the test session, not the real one:

```python
# conftest.py
from app.database import get_db
from app.main import app

@pytest.fixture
async def db_session(engine):
    async with async_sessionmaker(engine, expire_on_commit=False)() as session:
        # AIDEV-NOTE: override FastAPI DI so all route handlers use the test session
        app.dependency_overrides[get_db] = lambda: session
        yield session
        app.dependency_overrides.clear()
        await session.rollback()
```

---

## Common Rules

| Rule | Detail |
|------|--------|
| **Always Docker** | Postgres runs in a container locally — never `brew install postgresql` |
| **Real DB in tests** | Integration tests hit a real Postgres, never SQLite or mocks |
| **Separate test DB** | Use a dedicated database for tests (`app_test`) to avoid clobbering dev data |
| **Migrations over raw SQL** | All schema changes go through Alembic (Python) or Prisma Migrate (TS) |
| **Connection pooling** | Set `pool_size`/`max_overflow` (Python) or use PgBouncer for prod |
| **Commit lock files** | `uv.lock` and `package-lock.json` ensure reproducible dependency resolution |
