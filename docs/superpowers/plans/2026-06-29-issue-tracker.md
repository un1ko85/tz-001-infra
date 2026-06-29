# Issue Tracker Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a fullstack helpdesk issue tracker with FastAPI backend, React frontend, and Docker infrastructure.

**Architecture:** Clean Architecture backend (routers → services → models) with async SQLAlchemy. Feature-sliced frontend (auth, requests, admin) with Zustand for auth state and TanStack Query for server state. Three independent Git repos orchestrated by Docker Compose.

**Tech Stack:** Python 3.12, FastAPI, SQLAlchemy 2.0 (async), aiosqlite, Alembic, Pytest, httpx | React 18, TypeScript, Vite, Tailwind CSS v3, shadcn/ui, TanStack Query v5, Zustand, Axios, React Router v6, Vitest | Docker, Docker Compose, GitHub Actions

## Global Constraints

- Python 3.12+, Node.js 20+
- All backend code is async (`async def`, `AsyncSession`)
- SQLite via `aiosqlite` — no PostgreSQL-specific features
- All IDs are Integer auto-increment
- JWT access token: 15 min, refresh token: 7 days (httpOnly cookie)
- bcrypt password hashing via `passlib[bcrypt]`
- Priority weights: `low=0`, `normal=1`, `high=2`
- Offset-based pagination: `?page=1&per_page=20`
- Backend root path: `backend/`
- Frontend root path: `frontend/`
- Infrastructure root path: `./` (workspace root `tz-github/`)
- Every task commits at the end
- TDD: write failing test first, then implement

## File Map

### Backend (`backend/`)

| File | Responsibility |
|------|---------------|
| `pyproject.toml` | Dependencies, project metadata, ruff/pytest config |
| `alembic.ini` | Alembic config pointing to async DB |
| `alembic/env.py` | Async migration runner |
| `alembic/versions/` | Auto-generated migrations |
| `app/__init__.py` | Package marker |
| `app/main.py` | FastAPI app factory, lifespan (seed), CORS middleware |
| `app/config.py` | Pydantic BaseSettings (env vars) |
| `app/database.py` | Async engine, async_sessionmaker, Base |
| `app/models/__init__.py` | Re-exports User, Request |
| `app/models/user.py` | User SQLAlchemy model with role enum |
| `app/models/request.py` | Request SQLAlchemy model with status/priority enums |
| `app/schemas/__init__.py` | Package marker |
| `app/schemas/auth.py` | TokenResponse |
| `app/schemas/user.py` | UserCreate, UserResponse, RoleUpdate |
| `app/schemas/request.py` | RequestCreate, RequestUpdate, RequestResponse, RequestListResponse |
| `app/api/__init__.py` | api_router combining all sub-routers |
| `app/api/auth.py` | POST register, login, refresh, logout |
| `app/api/users.py` | GET /users/me, GET /users, PATCH /users/{id}/role |
| `app/api/requests.py` | GET/POST/PATCH/DELETE /requests |
| `app/services/__init__.py` | Package marker |
| `app/services/auth.py` | hash_password, verify_password, create_access_token, create_refresh_token, decode_token |
| `app/services/user.py` | get_user_by_username, get_user_by_id, list_users, update_role |
| `app/services/request.py` | create_request, get_request, list_requests, update_request, delete_request |
| `app/dependencies/__init__.py` | Package marker |
| `app/dependencies/auth.py` | get_current_user, require_role |
| `app/dependencies/database.py` | get_db async session generator |
| `app/seed.py` | create_default_admin |
| `tests/conftest.py` | Async fixtures: engine, session, client, auth helpers |
| `tests/test_auth.py` | Auth endpoint tests |
| `tests/test_requests.py` | Request CRUD tests |
| `tests/test_permissions.py` | Permission matrix tests |

### Frontend (`frontend/`)

| File | Responsibility |
|------|---------------|
| `package.json` | Dependencies, scripts |
| `vite.config.ts` | Vite config with proxy |
| `tsconfig.json` | TypeScript strict config |
| `tailwind.config.ts` | Tailwind CSS config |
| `components.json` | shadcn/ui config |
| `postcss.config.js` | PostCSS for Tailwind |
| `index.html` | HTML entry |
| `src/main.tsx` | React DOM render |
| `src/App.tsx` | QueryClientProvider + RouterProvider |
| `src/router.tsx` | Route definitions with guards |
| `src/shared/api/client.ts` | Axios instance + interceptors |
| `src/shared/api/types.ts` | PaginatedResponse, ApiError, User, Request types |
| `src/shared/lib/utils.ts` | cn() helper |
| `src/shared/hooks/use-debounce.ts` | Debounce hook |
| `src/shared/ui/` | shadcn/ui components |
| `src/features/auth/store/auth.store.ts` | Zustand auth store |
| `src/features/auth/api/auth.api.ts` | Auth API functions |
| `src/features/auth/api/auth.queries.ts` | Auth mutation hooks |
| `src/features/auth/ui/LoginForm.tsx` | Login form component |
| `src/features/auth/ui/RegisterForm.tsx` | Register form component |
| `src/features/auth/guards/ProtectedRoute.tsx` | Auth guard |
| `src/features/auth/guards/GuestRoute.tsx` | Guest guard |
| `src/features/requests/api/requests.api.ts` | Request API functions |
| `src/features/requests/api/requests.queries.ts` | Request query/mutation hooks |
| `src/features/requests/types.ts` | Request-specific types |
| `src/features/requests/ui/RequestList.tsx` | Request list with pagination |
| `src/features/requests/ui/RequestCard.tsx` | Single request card |
| `src/features/requests/ui/RequestFilters.tsx` | Search + filter controls |
| `src/features/requests/ui/RequestForm.tsx` | Create/Edit dialog |
| `src/features/requests/ui/RequestStatusBadge.tsx` | Status badge |
| `src/features/admin/api/users.api.ts` | Admin user API functions |
| `src/features/admin/api/users.queries.ts` | Admin query hooks |
| `src/features/admin/ui/UserList.tsx` | User management table |
| `src/features/admin/ui/RoleSelect.tsx` | Role dropdown |
| `src/pages/LoginPage.tsx` | Login page |
| `src/pages/RegisterPage.tsx` | Register page |
| `src/pages/DashboardPage.tsx` | Main dashboard |
| `src/pages/AdminPage.tsx` | Admin panel |
| `src/styles/globals.css` | Global styles + Tailwind directives |
| `nginx.conf` | Production nginx config |
| `Dockerfile` | Multi-stage production build |

### Infrastructure (`./`)

| File | Responsibility |
|------|---------------|
| `docker-compose.yml` | Production services |
| `docker-compose.dev.yml` | Development services with hot-reload |
| `.env.example` | Template env vars |

---

## Task 1: Backend Project Skeleton + Config + Database

**Files:**
- Create: `backend/pyproject.toml`
- Create: `backend/app/__init__.py`
- Create: `backend/app/config.py`
- Create: `backend/app/database.py`
- Create: `backend/app/main.py`
- Test: `backend/tests/conftest.py`
- Test: `backend/tests/test_health.py`

**Interfaces:**
- Consumes: nothing (first task)
- Produces:
  - `app.config.Settings` — Pydantic BaseSettings with `DATABASE_URL: str`, `JWT_SECRET: str`, `JWT_ACCESS_EXPIRE_MINUTES: int = 15`, `JWT_REFRESH_EXPIRE_DAYS: int = 7`, `CORS_ORIGINS: str = "http://localhost:5173"`, `ADMIN_USERNAME: str = "admin"`, `ADMIN_PASSWORD: str = "admin"`
  - `app.config.get_settings() -> Settings` — cached settings getter
  - `app.database.Base` — SQLAlchemy declarative base
  - `app.database.async_engine` — async engine from `DATABASE_URL`
  - `app.database.async_session_maker` — `async_sessionmaker[AsyncSession]`
  - `app.main.app` — FastAPI application instance with CORS and `/api/health` endpoint

- [ ] **Step 1: Create `pyproject.toml` with all dependencies**

```toml
[project]
name = "issue-tracker-backend"
version = "0.1.0"
description = "Backend for Issue Tracker helpdesk system"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.32.0",
    "sqlalchemy[asyncio]>=2.0.36",
    "aiosqlite>=0.20.0",
    "alembic>=1.14.0",
    "python-jose[cryptography]>=3.3.0",
    "passlib[bcrypt]>=1.7.4",
    "python-multipart>=0.0.12",
    "pydantic-settings>=2.6.0",
    "httpx>=0.28.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.3.0",
    "pytest-asyncio>=0.24.0",
    "httpx>=0.28.0",
    "ruff>=0.8.0",
]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]

[tool.ruff]
target-version = "py312"
line-length = 120

[tool.ruff.lint]
select = ["E", "F", "I", "UP"]

[build-system]
requires = ["setuptools>=75.0"]
build-backend = "setuptools.build_meta"
```

- [ ] **Step 2: Create `app/__init__.py`**

```python
```

(Empty file — package marker.)

- [ ] **Step 3: Create `app/config.py`**

```python
from functools import lru_cache

from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    DATABASE_URL: str = "sqlite+aiosqlite:///./data/app.db"
    JWT_SECRET: str = "change-me-in-production"
    JWT_ACCESS_EXPIRE_MINUTES: int = 15
    JWT_REFRESH_EXPIRE_DAYS: int = 7
    CORS_ORIGINS: str = "http://localhost:5173"
    ADMIN_USERNAME: str = "admin"
    ADMIN_PASSWORD: str = "admin"

    model_config = {"env_file": ".env", "extra": "ignore"}


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

- [ ] **Step 4: Create `app/database.py`**

```python
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase

from app.config import get_settings

settings = get_settings()

async_engine = create_async_engine(settings.DATABASE_URL, echo=False)
async_session_maker = async_sessionmaker(async_engine, class_=AsyncSession, expire_on_commit=False)


class Base(DeclarativeBase):
    pass
```

- [ ] **Step 5: Create `app/main.py` with health endpoint**

```python
from contextlib import asynccontextmanager

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.config import get_settings


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: seed admin user (will be added in Task 3)
    yield


def create_app() -> FastAPI:
    settings = get_settings()
    application = FastAPI(title="Issue Tracker API", lifespan=lifespan)

    application.add_middleware(
        CORSMiddleware,
        allow_origins=settings.CORS_ORIGINS.split(","),
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    @application.get("/api/health")
    async def health():
        return {"status": "ok"}

    return application


app = create_app()
```

- [ ] **Step 6: Create test fixtures in `tests/conftest.py`**

```python
import pytest
from httpx import ASGITransport, AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from app.database import Base
from app.main import app


@pytest.fixture
async def async_engine():
    engine = create_async_engine("sqlite+aiosqlite:///:memory:", echo=False)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()


@pytest.fixture
async def db_session(async_engine):
    session_maker = async_sessionmaker(async_engine, class_=AsyncSession, expire_on_commit=False)
    async with session_maker() as session:
        yield session


@pytest.fixture
async def client():
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac
```

- [ ] **Step 7: Write health endpoint test in `tests/test_health.py`**

```python
import pytest


@pytest.mark.asyncio
async def test_health_endpoint(client):
    response = await client.get("/api/health")
    assert response.status_code == 200
    assert response.json() == {"status": "ok"}
```

- [ ] **Step 8: Install dependencies and run test**

```bash
cd backend
pip install -e ".[dev]"
pytest tests/test_health.py -v
```

Expected: PASS — `test_health_endpoint PASSED`

- [ ] **Step 9: Commit**

```bash
cd backend
git add .
git commit -m "feat: project skeleton with config, database, and health endpoint"
```

---

## Task 2: Backend Models + Alembic Migrations

**Files:**
- Create: `backend/app/models/__init__.py`
- Create: `backend/app/models/user.py`
- Create: `backend/app/models/request.py`
- Create: `backend/alembic.ini`
- Create: `backend/alembic/env.py`
- Create: `backend/alembic/script.py.mako`
- Test: `backend/tests/test_models.py`

**Interfaces:**
- Consumes: `app.database.Base` from Task 1
- Produces:
  - `app.models.user.UserRole` — enum with values `user`, `manager`, `admin`
  - `app.models.user.User` — SQLAlchemy model with columns: `id: int`, `username: str`, `password_hash: str`, `role: UserRole`, `created_at: datetime`
  - `app.models.request.RequestStatus` — enum with values `new`, `in_progress`, `done`
  - `app.models.request.RequestPriority` — enum with values `low`, `normal`, `high`; attribute `weight: int` (low=0, normal=1, high=2)
  - `app.models.request.Request` — SQLAlchemy model with columns: `id: int`, `title: str`, `description: str | None`, `status: RequestStatus`, `priority: RequestPriority`, `author_id: int`, `assignee_id: int | None`, `created_at: datetime`, `updated_at: datetime`

- [ ] **Step 1: Create `app/models/user.py`**

```python
import enum
from datetime import datetime, timezone

from sqlalchemy import DateTime, Enum, Integer, String, func
from sqlalchemy.orm import Mapped, mapped_column

from app.database import Base


class UserRole(str, enum.Enum):
    user = "user"
    manager = "manager"
    admin = "admin"


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    username: Mapped[str] = mapped_column(String(50), unique=True, nullable=False, index=True)
    password_hash: Mapped[str] = mapped_column(String(128), nullable=False)
    role: Mapped[UserRole] = mapped_column(Enum(UserRole), default=UserRole.user, nullable=False)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), nullable=False
    )
```

- [ ] **Step 2: Create `app/models/request.py`**

```python
import enum
from datetime import datetime

from sqlalchemy import DateTime, Enum, ForeignKey, Integer, String, Text, func
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.database import Base


class RequestStatus(str, enum.Enum):
    new = "new"
    in_progress = "in_progress"
    done = "done"


class RequestPriority(str, enum.Enum):
    low = "low"
    normal = "normal"
    high = "high"

    @property
    def weight(self) -> int:
        return {"low": 0, "normal": 1, "high": 2}[self.value]


class Request(Base):
    __tablename__ = "requests"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    title: Mapped[str] = mapped_column(String(120), nullable=False)
    description: Mapped[str | None] = mapped_column(Text, nullable=True)
    status: Mapped[RequestStatus] = mapped_column(
        Enum(RequestStatus), default=RequestStatus.new, nullable=False, index=True
    )
    priority: Mapped[RequestPriority] = mapped_column(
        Enum(RequestPriority), default=RequestPriority.normal, nullable=False, index=True
    )
    author_id: Mapped[int] = mapped_column(Integer, ForeignKey("users.id"), nullable=False, index=True)
    assignee_id: Mapped[int | None] = mapped_column(Integer, ForeignKey("users.id"), nullable=True)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), nullable=False, index=True
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), onupdate=func.now(), nullable=False
    )

    author = relationship("User", foreign_keys=[author_id], lazy="selectin")
    assignee = relationship("User", foreign_keys=[assignee_id], lazy="selectin")
```

- [ ] **Step 3: Create `app/models/__init__.py`**

```python
from app.models.request import Request, RequestPriority, RequestStatus
from app.models.user import User, UserRole

__all__ = ["User", "UserRole", "Request", "RequestStatus", "RequestPriority"]
```

- [ ] **Step 4: Write model test in `tests/test_models.py`**

```python
import pytest
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.user import User, UserRole
from app.models.request import Request, RequestStatus, RequestPriority


@pytest.mark.asyncio
async def test_create_user(db_session: AsyncSession):
    user = User(username="testuser", password_hash="fakehash", role=UserRole.user)
    db_session.add(user)
    await db_session.commit()

    result = await db_session.execute(select(User).where(User.username == "testuser"))
    saved = result.scalar_one()
    assert saved.id is not None
    assert saved.username == "testuser"
    assert saved.role == UserRole.user
    assert saved.created_at is not None


@pytest.mark.asyncio
async def test_create_request(db_session: AsyncSession):
    user = User(username="author", password_hash="fakehash")
    db_session.add(user)
    await db_session.commit()

    req = Request(title="Test Request", description="A test", author_id=user.id)
    db_session.add(req)
    await db_session.commit()

    result = await db_session.execute(select(Request).where(Request.id == req.id))
    saved = result.scalar_one()
    assert saved.title == "Test Request"
    assert saved.status == RequestStatus.new
    assert saved.priority == RequestPriority.normal
    assert saved.author_id == user.id
    assert saved.created_at is not None


@pytest.mark.asyncio
async def test_priority_weights():
    assert RequestPriority.low.weight == 0
    assert RequestPriority.normal.weight == 1
    assert RequestPriority.high.weight == 2
```

- [ ] **Step 5: Run model tests**

```bash
cd backend
pytest tests/test_models.py -v
```

Expected: 3 tests PASS

- [ ] **Step 6: Initialize Alembic**

Create `alembic.ini`:

```ini
[alembic]
script_location = alembic
sqlalchemy.url = sqlite+aiosqlite:///./data/app.db

[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARN
handlers = console

[logger_sqlalchemy]
level = WARN
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
datefmt = %H:%M:%S
```

Create `alembic/env.py`:

```python
import asyncio
from logging.config import fileConfig

from alembic import context
from sqlalchemy.ext.asyncio import create_async_engine

from app.config import get_settings
from app.database import Base
from app.models import User, Request  # noqa: F401 — ensure models are registered

config = context.config
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata


def run_migrations_offline() -> None:
    url = get_settings().DATABASE_URL
    context.configure(url=url, target_metadata=target_metadata, literal_binds=True)
    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection):
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


async def run_migrations_online() -> None:
    engine = create_async_engine(get_settings().DATABASE_URL)
    async with engine.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await engine.dispose()


if context.is_offline_mode():
    run_migrations_offline()
else:
    asyncio.run(run_migrations_online())
```

Create `alembic/script.py.mako`:

```mako
"""${message}

Revision ID: ${up_revision}
Revises: ${down_revision | comma,n}
Create Date: ${create_date}
"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa
${imports if imports else ""}

revision: str = ${repr(up_revision)}
down_revision: Union[str, None] = ${repr(down_revision)}
branch_labels: Union[str, Sequence[str], None] = ${repr(branch_labels)}
depends_on: Union[str, Sequence[str], None] = ${repr(depends_on)}


def upgrade() -> None:
    ${upgrades if upgrades else "pass"}


def downgrade() -> None:
    ${downgrades if downgrades else "pass"}
```

- [ ] **Step 7: Generate initial migration**

```bash
cd backend
mkdir -p data
alembic revision --autogenerate -m "initial: users and requests tables"
```

Expected: Migration file created in `alembic/versions/`

- [ ] **Step 8: Commit**

```bash
cd backend
git add .
git commit -m "feat: User and Request models with Alembic migrations"
```

---

## Task 3: Backend Auth Service + Auth Schemas + Seed

**Files:**
- Create: `backend/app/schemas/__init__.py`
- Create: `backend/app/schemas/auth.py`
- Create: `backend/app/schemas/user.py`
- Create: `backend/app/services/__init__.py`
- Create: `backend/app/services/auth.py`
- Create: `backend/app/dependencies/__init__.py`
- Create: `backend/app/dependencies/database.py`
- Create: `backend/app/dependencies/auth.py`
- Create: `backend/app/seed.py`
- Modify: `backend/app/main.py` (add seed to lifespan)
- Modify: `backend/tests/conftest.py` (add auth helper fixtures)
- Test: `backend/tests/test_auth_service.py`

**Interfaces:**
- Consumes: `app.models.User`, `app.models.UserRole`, `app.database.async_session_maker`, `app.config.Settings`
- Produces:
  - `app.schemas.auth.TokenResponse` — `access_token: str`, `token_type: str = "bearer"`, `user: UserResponse`
  - `app.schemas.user.UserCreate` — `username: str (3-50)`, `password: str (6+)`
  - `app.schemas.user.UserResponse` — `id: int`, `username: str`, `role: UserRole`, `created_at: datetime`
  - `app.schemas.user.RoleUpdate` — `role: UserRole`
  - `app.services.auth.hash_password(password: str) -> str`
  - `app.services.auth.verify_password(plain: str, hashed: str) -> bool`
  - `app.services.auth.create_access_token(user_id: int, role: str) -> str`
  - `app.services.auth.create_refresh_token(user_id: int) -> str`
  - `app.services.auth.decode_token(token: str) -> dict`
  - `app.dependencies.auth.get_current_user(token, db) -> User`
  - `app.dependencies.auth.require_role(*roles) -> Callable`
  - `app.dependencies.database.get_db() -> AsyncGenerator[AsyncSession]`
  - `app.seed.create_default_admin(session: AsyncSession) -> None`
  - Test fixture: `conftest.create_test_user(db_session, username, password, role) -> User`
  - Test fixture: `conftest.get_auth_headers(client, username, password) -> dict`

- [ ] **Step 1: Create `app/schemas/__init__.py`**

```python
```

- [ ] **Step 2: Create `app/schemas/user.py`**

```python
from datetime import datetime

from pydantic import BaseModel, Field

from app.models.user import UserRole


class UserCreate(BaseModel):
    username: str = Field(min_length=3, max_length=50, pattern=r"^[a-zA-Z0-9_]+$")
    password: str = Field(min_length=6)


class UserResponse(BaseModel):
    id: int
    username: str
    role: UserRole
    created_at: datetime

    model_config = {"from_attributes": True}


class RoleUpdate(BaseModel):
    role: UserRole
```

- [ ] **Step 3: Create `app/schemas/auth.py`**

```python
from pydantic import BaseModel

from app.schemas.user import UserResponse


class TokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"
    user: UserResponse | None = None
```

- [ ] **Step 4: Create `app/services/__init__.py`**

```python
```

- [ ] **Step 5: Create `app/services/auth.py`**

```python
from datetime import datetime, timedelta, timezone

from jose import JWTError, jwt
from passlib.context import CryptContext

from app.config import get_settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


def hash_password(password: str) -> str:
    return pwd_context.hash(password)


def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)


def create_access_token(user_id: int, role: str) -> str:
    settings = get_settings()
    expire = datetime.now(timezone.utc) + timedelta(minutes=settings.JWT_ACCESS_EXPIRE_MINUTES)
    payload = {"sub": str(user_id), "role": role, "exp": expire, "type": "access"}
    return jwt.encode(payload, settings.JWT_SECRET, algorithm="HS256")


def create_refresh_token(user_id: int) -> str:
    settings = get_settings()
    expire = datetime.now(timezone.utc) + timedelta(days=settings.JWT_REFRESH_EXPIRE_DAYS)
    payload = {"sub": str(user_id), "exp": expire, "type": "refresh"}
    return jwt.encode(payload, settings.JWT_SECRET, algorithm="HS256")


def decode_token(token: str) -> dict:
    settings = get_settings()
    try:
        payload = jwt.decode(token, settings.JWT_SECRET, algorithms=["HS256"])
        return payload
    except JWTError:
        return {}
```

- [ ] **Step 6: Create `app/dependencies/__init__.py`, `app/dependencies/database.py`**

`app/dependencies/__init__.py`:
```python
```

`app/dependencies/database.py`:
```python
from collections.abc import AsyncGenerator

from sqlalchemy.ext.asyncio import AsyncSession

from app.database import async_session_maker


async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_maker() as session:
        yield session
```

- [ ] **Step 7: Create `app/dependencies/auth.py`**

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.dependencies.database import get_db
from app.models.user import User, UserRole
from app.services.auth import decode_token

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/auth/login")


async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    payload = decode_token(token)
    user_id = payload.get("sub")
    token_type = payload.get("type")

    if not user_id or token_type != "access":
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid or expired token")

    result = await db.execute(select(User).where(User.id == int(user_id)))
    user = result.scalar_one_or_none()

    if user is None:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="User not found")

    return user


def require_role(*roles: UserRole):
    async def role_checker(current_user: User = Depends(get_current_user)) -> User:
        if current_user.role not in roles:
            raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Insufficient permissions")
        return current_user

    return role_checker
```

- [ ] **Step 8: Create `app/seed.py`**

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.config import get_settings
from app.models.user import User, UserRole
from app.services.auth import hash_password


async def create_default_admin(session: AsyncSession) -> None:
    settings = get_settings()
    result = await session.execute(select(User).where(User.username == settings.ADMIN_USERNAME))
    if result.scalar_one_or_none() is not None:
        return

    admin = User(
        username=settings.ADMIN_USERNAME,
        password_hash=hash_password(settings.ADMIN_PASSWORD),
        role=UserRole.admin,
    )
    session.add(admin)
    await session.commit()
```

- [ ] **Step 9: Update `app/main.py` lifespan to call seed**

```python
from contextlib import asynccontextmanager

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.config import get_settings
from app.database import Base, async_engine, async_session_maker
from app.seed import create_default_admin


@asynccontextmanager
async def lifespan(app: FastAPI):
    async with async_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    async with async_session_maker() as session:
        await create_default_admin(session)
    yield


def create_app() -> FastAPI:
    settings = get_settings()
    application = FastAPI(title="Issue Tracker API", lifespan=lifespan)

    application.add_middleware(
        CORSMiddleware,
        allow_origins=settings.CORS_ORIGINS.split(","),
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    @application.get("/api/health")
    async def health():
        return {"status": "ok"}

    return application


app = create_app()
```

- [ ] **Step 10: Write auth service unit tests in `tests/test_auth_service.py`**

```python
import pytest

from app.services.auth import (
    create_access_token,
    create_refresh_token,
    decode_token,
    hash_password,
    verify_password,
)


def test_hash_and_verify_password():
    hashed = hash_password("mypassword")
    assert hashed != "mypassword"
    assert verify_password("mypassword", hashed) is True
    assert verify_password("wrongpassword", hashed) is False


def test_create_and_decode_access_token():
    token = create_access_token(user_id=1, role="user")
    payload = decode_token(token)
    assert payload["sub"] == "1"
    assert payload["role"] == "user"
    assert payload["type"] == "access"


def test_create_and_decode_refresh_token():
    token = create_refresh_token(user_id=42)
    payload = decode_token(token)
    assert payload["sub"] == "42"
    assert payload["type"] == "refresh"
    assert "role" not in payload


def test_decode_invalid_token():
    payload = decode_token("invalid.token.here")
    assert payload == {}
```

- [ ] **Step 11: Update `tests/conftest.py` with auth helper fixtures**

Add to the existing `conftest.py`:

```python
from app.models.user import User, UserRole
from app.services.auth import hash_password
from app.dependencies.database import get_db


@pytest.fixture
async def override_get_db(async_engine):
    """Override the get_db dependency to use test database."""
    from sqlalchemy.ext.asyncio import async_sessionmaker, AsyncSession

    test_session_maker = async_sessionmaker(async_engine, class_=AsyncSession, expire_on_commit=False)

    async def _override():
        async with test_session_maker() as session:
            yield session

    app.dependency_overrides[get_db] = _override
    yield
    app.dependency_overrides.clear()


@pytest.fixture
async def client(override_get_db):
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac


async def create_test_user(
    session: AsyncSession, username: str = "testuser", password: str = "testpass", role: UserRole = UserRole.user
) -> User:
    user = User(username=username, password_hash=hash_password(password), role=role)
    session.add(user)
    await session.commit()
    await session.refresh(user)
    return user
```

- [ ] **Step 12: Run all tests**

```bash
cd backend
pytest -v
```

Expected: All tests PASS

- [ ] **Step 13: Commit**

```bash
cd backend
git add .
git commit -m "feat: auth service, schemas, dependencies, seed, and test fixtures"
```

---

## Task 4: Backend Auth Endpoints (Register, Login, Refresh, Logout)

**Files:**
- Create: `backend/app/services/user.py`
- Create: `backend/app/api/__init__.py`
- Create: `backend/app/api/auth.py`
- Modify: `backend/app/main.py` (include api_router)
- Test: `backend/tests/test_auth.py`

**Interfaces:**
- Consumes: `app.services.auth.*`, `app.schemas.user.UserCreate`, `app.schemas.auth.TokenResponse`, `app.dependencies.database.get_db`, `app.models.User`
- Produces:
  - `app.services.user.get_user_by_username(db, username) -> User | None`
  - `app.services.user.get_user_by_id(db, user_id) -> User | None`
  - `app.services.user.create_user(db, username, password) -> User`
  - `POST /api/auth/register` — creates user, returns 201 UserResponse
  - `POST /api/auth/login` — OAuth2 form, returns TokenResponse + Set-Cookie refresh
  - `POST /api/auth/refresh` — reads refresh cookie, returns new TokenResponse
  - `POST /api/auth/logout` — clears refresh cookie

- [ ] **Step 1: Create `app/services/user.py`**

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.user import User, UserRole
from app.services.auth import hash_password


async def get_user_by_username(db: AsyncSession, username: str) -> User | None:
    result = await db.execute(select(User).where(User.username == username))
    return result.scalar_one_or_none()


async def get_user_by_id(db: AsyncSession, user_id: int) -> User | None:
    result = await db.execute(select(User).where(User.id == user_id))
    return result.scalar_one_or_none()


async def create_user(db: AsyncSession, username: str, password: str) -> User:
    user = User(username=username, password_hash=hash_password(password), role=UserRole.user)
    db.add(user)
    await db.commit()
    await db.refresh(user)
    return user


async def list_users(db: AsyncSession) -> list[User]:
    result = await db.execute(select(User).order_by(User.id))
    return list(result.scalars().all())


async def update_user_role(db: AsyncSession, user: User, role: UserRole) -> User:
    user.role = role
    await db.commit()
    await db.refresh(user)
    return user
```

- [ ] **Step 2: Create `app/api/auth.py`**

```python
from fastapi import APIRouter, Depends, HTTPException, Response, Cookie, status
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy.ext.asyncio import AsyncSession

from app.dependencies.auth import get_current_user
from app.dependencies.database import get_db
from app.models.user import User
from app.schemas.auth import TokenResponse
from app.schemas.user import UserCreate, UserResponse
from app.services.auth import create_access_token, create_refresh_token, decode_token, verify_password
from app.services.user import create_user, get_user_by_username

router = APIRouter(prefix="/auth", tags=["auth"])

REFRESH_COOKIE_KEY = "refresh_token"
REFRESH_MAX_AGE = 7 * 24 * 60 * 60  # 7 days in seconds


@router.post("/register", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def register(data: UserCreate, db: AsyncSession = Depends(get_db)):
    existing = await get_user_by_username(db, data.username)
    if existing:
        raise HTTPException(status_code=status.HTTP_409_CONFLICT, detail="Username already taken")
    user = await create_user(db, data.username, data.password)
    return user


@router.post("/login", response_model=TokenResponse)
async def login(response: Response, form: OAuth2PasswordRequestForm = Depends(), db: AsyncSession = Depends(get_db)):
    user = await get_user_by_username(db, form.username)
    if not user or not verify_password(form.password, user.password_hash):
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid credentials")

    access_token = create_access_token(user.id, user.role.value)
    refresh_token = create_refresh_token(user.id)

    response.set_cookie(
        key=REFRESH_COOKIE_KEY,
        value=refresh_token,
        httponly=True,
        secure=False,  # Set True in production with HTTPS
        samesite="lax",
        max_age=REFRESH_MAX_AGE,
        path="/api/auth",
    )

    return TokenResponse(access_token=access_token, user=UserResponse.model_validate(user))


@router.post("/refresh", response_model=TokenResponse)
async def refresh(response: Response, refresh_token: str | None = Cookie(None), db: AsyncSession = Depends(get_db)):
    if not refresh_token:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Refresh token missing")

    payload = decode_token(refresh_token)
    user_id = payload.get("sub")
    token_type = payload.get("type")

    if not user_id or token_type != "refresh":
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid refresh token")

    from app.services.user import get_user_by_id

    user = await get_user_by_id(db, int(user_id))
    if not user:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="User not found")

    new_access = create_access_token(user.id, user.role.value)
    new_refresh = create_refresh_token(user.id)

    response.set_cookie(
        key=REFRESH_COOKIE_KEY,
        value=new_refresh,
        httponly=True,
        secure=False,
        samesite="lax",
        max_age=REFRESH_MAX_AGE,
        path="/api/auth",
    )

    return TokenResponse(access_token=new_access)


@router.post("/logout")
async def logout(response: Response, _: User = Depends(get_current_user)):
    response.delete_cookie(key=REFRESH_COOKIE_KEY, path="/api/auth")
    return {"detail": "logged out"}
```

- [ ] **Step 3: Create `app/api/__init__.py` and include router**

```python
from fastapi import APIRouter

from app.api.auth import router as auth_router

api_router = APIRouter(prefix="/api")
api_router.include_router(auth_router)
```

- [ ] **Step 4: Update `app/main.py` to include `api_router`**

Add import and include after CORS middleware:

```python
from app.api import api_router
# ... inside create_app(), after add_middleware:
application.include_router(api_router)
```

- [ ] **Step 5: Write auth endpoint tests in `tests/test_auth.py`**

```python
import pytest
from httpx import AsyncClient


@pytest.mark.asyncio
async def test_register_success(client: AsyncClient):
    response = await client.post("/api/auth/register", json={"username": "newuser", "password": "secret123"})
    assert response.status_code == 201
    data = response.json()
    assert data["username"] == "newuser"
    assert data["role"] == "user"
    assert "id" in data


@pytest.mark.asyncio
async def test_register_duplicate(client: AsyncClient):
    await client.post("/api/auth/register", json={"username": "dupuser", "password": "secret123"})
    response = await client.post("/api/auth/register", json={"username": "dupuser", "password": "secret123"})
    assert response.status_code == 409


@pytest.mark.asyncio
async def test_register_validation_short_username(client: AsyncClient):
    response = await client.post("/api/auth/register", json={"username": "ab", "password": "secret123"})
    assert response.status_code == 422


@pytest.mark.asyncio
async def test_register_validation_short_password(client: AsyncClient):
    response = await client.post("/api/auth/register", json={"username": "validuser", "password": "12345"})
    assert response.status_code == 422


@pytest.mark.asyncio
async def test_login_success(client: AsyncClient):
    await client.post("/api/auth/register", json={"username": "loginuser", "password": "secret123"})
    response = await client.post("/api/auth/login", data={"username": "loginuser", "password": "secret123"})
    assert response.status_code == 200
    data = response.json()
    assert "access_token" in data
    assert data["token_type"] == "bearer"
    assert data["user"]["username"] == "loginuser"
    assert "refresh_token" in response.cookies


@pytest.mark.asyncio
async def test_login_wrong_password(client: AsyncClient):
    await client.post("/api/auth/register", json={"username": "wrongpw", "password": "secret123"})
    response = await client.post("/api/auth/login", data={"username": "wrongpw", "password": "wrongpassword"})
    assert response.status_code == 401


@pytest.mark.asyncio
async def test_login_nonexistent_user(client: AsyncClient):
    response = await client.post("/api/auth/login", data={"username": "noexist", "password": "secret123"})
    assert response.status_code == 401


@pytest.mark.asyncio
async def test_refresh_token(client: AsyncClient):
    await client.post("/api/auth/register", json={"username": "refreshuser", "password": "secret123"})
    login_resp = await client.post("/api/auth/login", data={"username": "refreshuser", "password": "secret123"})
    cookies = login_resp.cookies

    response = await client.post("/api/auth/refresh", cookies=cookies)
    assert response.status_code == 200
    assert "access_token" in response.json()


@pytest.mark.asyncio
async def test_refresh_without_cookie(client: AsyncClient):
    response = await client.post("/api/auth/refresh")
    assert response.status_code == 401


@pytest.mark.asyncio
async def test_logout(client: AsyncClient):
    await client.post("/api/auth/register", json={"username": "logoutuser", "password": "secret123"})
    login_resp = await client.post("/api/auth/login", data={"username": "logoutuser", "password": "secret123"})
    token = login_resp.json()["access_token"]

    response = await client.post("/api/auth/logout", headers={"Authorization": f"Bearer {token}"})
    assert response.status_code == 200
    assert response.json()["detail"] == "logged out"
```

- [ ] **Step 6: Run all tests**

```bash
cd backend
pytest -v
```

Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
cd backend
git add .
git commit -m "feat: auth endpoints — register, login, refresh, logout"
```

---

## Task 5: Backend User Endpoints + Request Schemas

**Files:**
- Create: `backend/app/api/users.py`
- Create: `backend/app/schemas/request.py`
- Modify: `backend/app/api/__init__.py` (include users router)
- Test: `backend/tests/test_users.py`

**Interfaces:**
- Consumes: `app.services.user.*`, `app.dependencies.auth.*`, `app.models.UserRole`
- Produces:
  - `GET /api/users/me` — returns current user
  - `GET /api/users` — admin only, returns all users
  - `PATCH /api/users/{id}/role` — admin only, updates user role
  - `app.schemas.request.RequestCreate` — `title: str (3-120)`, `description: str | None (max 1000)`, `priority: RequestPriority = normal`
  - `app.schemas.request.RequestUpdate` — all fields optional: `title`, `description`, `priority`, `status`, `assignee_id`
  - `app.schemas.request.RequestResponse` — full request with author/assignee names
  - `app.schemas.request.RequestListResponse` — `items: list[RequestResponse]`, `total: int`, `page: int`, `per_page: int`, `pages: int`

- [ ] **Step 1: Create `app/api/users.py`**

```python
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.dependencies.auth import get_current_user, require_role
from app.dependencies.database import get_db
from app.models.user import User, UserRole
from app.schemas.user import RoleUpdate, UserResponse
from app.services.user import get_user_by_id, list_users, update_user_role

router = APIRouter(prefix="/users", tags=["users"])


@router.get("/me", response_model=UserResponse)
async def get_me(current_user: User = Depends(get_current_user)):
    return current_user


@router.get("", response_model=list[UserResponse])
async def get_users(
    _: User = Depends(require_role(UserRole.admin)),
    db: AsyncSession = Depends(get_db),
):
    return await list_users(db)


@router.patch("/{user_id}/role", response_model=UserResponse)
async def change_role(
    user_id: int,
    data: RoleUpdate,
    current_user: User = Depends(require_role(UserRole.admin)),
    db: AsyncSession = Depends(get_db),
):
    if current_user.id == user_id:
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Cannot change your own role")

    user = await get_user_by_id(db, user_id)
    if not user:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="User not found")

    updated = await update_user_role(db, user, data.role)
    return updated
```

- [ ] **Step 2: Create `app/schemas/request.py`**

```python
from datetime import datetime

from pydantic import BaseModel, Field

from app.models.request import RequestPriority, RequestStatus


class RequestCreate(BaseModel):
    title: str = Field(min_length=3, max_length=120)
    description: str | None = Field(default=None, max_length=1000)
    priority: RequestPriority = RequestPriority.normal


class RequestUpdate(BaseModel):
    title: str | None = Field(default=None, min_length=3, max_length=120)
    description: str | None = Field(default=None, max_length=1000)
    priority: RequestPriority | None = None
    status: RequestStatus | None = None
    assignee_id: int | None = None


class AuthorResponse(BaseModel):
    id: int
    username: str

    model_config = {"from_attributes": True}


class RequestResponse(BaseModel):
    id: int
    title: str
    description: str | None
    status: RequestStatus
    priority: RequestPriority
    author: AuthorResponse
    assignee: AuthorResponse | None
    created_at: datetime
    updated_at: datetime

    model_config = {"from_attributes": True}


class RequestListResponse(BaseModel):
    items: list[RequestResponse]
    total: int
    page: int
    per_page: int
    pages: int
```

- [ ] **Step 3: Update `app/api/__init__.py` to include users router**

```python
from fastapi import APIRouter

from app.api.auth import router as auth_router
from app.api.users import router as users_router

api_router = APIRouter(prefix="/api")
api_router.include_router(auth_router)
api_router.include_router(users_router)
```

- [ ] **Step 4: Write user endpoint tests in `tests/test_users.py`**

```python
import pytest
from httpx import AsyncClient

from tests.conftest import create_test_user


@pytest.mark.asyncio
async def test_get_me(client: AsyncClient, db_session):
    await create_test_user(db_session, "meuser", "password123")
    login = await client.post("/api/auth/login", data={"username": "meuser", "password": "password123"})
    token = login.json()["access_token"]

    response = await client.get("/api/users/me", headers={"Authorization": f"Bearer {token}"})
    assert response.status_code == 200
    assert response.json()["username"] == "meuser"


@pytest.mark.asyncio
async def test_get_me_unauthorized(client: AsyncClient):
    response = await client.get("/api/users/me")
    assert response.status_code == 401


@pytest.mark.asyncio
async def test_list_users_admin_only(client: AsyncClient, db_session):
    from app.models.user import UserRole

    await create_test_user(db_session, "admin2", "password123", UserRole.admin)
    await create_test_user(db_session, "regular", "password123", UserRole.user)

    # Admin can list
    login = await client.post("/api/auth/login", data={"username": "admin2", "password": "password123"})
    token = login.json()["access_token"]
    response = await client.get("/api/users", headers={"Authorization": f"Bearer {token}"})
    assert response.status_code == 200
    assert len(response.json()) >= 2

    # Regular user cannot
    login = await client.post("/api/auth/login", data={"username": "regular", "password": "password123"})
    token = login.json()["access_token"]
    response = await client.get("/api/users", headers={"Authorization": f"Bearer {token}"})
    assert response.status_code == 403


@pytest.mark.asyncio
async def test_change_role(client: AsyncClient, db_session):
    from app.models.user import UserRole

    admin = await create_test_user(db_session, "roleadmin", "password123", UserRole.admin)
    user = await create_test_user(db_session, "targetuser", "password123", UserRole.user)

    login = await client.post("/api/auth/login", data={"username": "roleadmin", "password": "password123"})
    token = login.json()["access_token"]

    response = await client.patch(
        f"/api/users/{user.id}/role",
        json={"role": "manager"},
        headers={"Authorization": f"Bearer {token}"},
    )
    assert response.status_code == 200
    assert response.json()["role"] == "manager"


@pytest.mark.asyncio
async def test_cannot_change_own_role(client: AsyncClient, db_session):
    from app.models.user import UserRole

    admin = await create_test_user(db_session, "selfadmin", "password123", UserRole.admin)

    login = await client.post("/api/auth/login", data={"username": "selfadmin", "password": "password123"})
    token = login.json()["access_token"]

    response = await client.patch(
        f"/api/users/{admin.id}/role",
        json={"role": "user"},
        headers={"Authorization": f"Bearer {token}"},
    )
    assert response.status_code == 403
```

- [ ] **Step 5: Run all tests**

```bash
cd backend
pytest -v
```

Expected: All tests PASS

- [ ] **Step 6: Commit**

```bash
cd backend
git add .
git commit -m "feat: user endpoints (me, list, change role) and request schemas"
```

---

## Task 6: Backend Request CRUD + Filters + Permissions

**Files:**
- Create: `backend/app/services/request.py`
- Create: `backend/app/api/requests.py`
- Modify: `backend/app/api/__init__.py` (include requests router)
- Test: `backend/tests/test_requests.py`
- Test: `backend/tests/test_permissions.py`

**Interfaces:**
- Consumes: all previous services, schemas, dependencies
- Produces:
  - `app.services.request.create_request(db, title, description, priority, author_id) -> Request`
  - `app.services.request.get_request(db, request_id) -> Request | None`
  - `app.services.request.list_requests(db, author_id, search, status, priority, sort_by, sort_order, page, per_page) -> tuple[list[Request], int]`
  - `app.services.request.update_request(db, request, data, current_user) -> Request` — raises HTTPException on permission violations
  - `app.services.request.delete_request(db, request_id) -> None`
  - Full CRUD endpoints at `/api/requests`

- [ ] **Step 1: Create `app/services/request.py`**

```python
import math

from fastapi import HTTPException, status
from sqlalchemy import Select, func, select, case
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.request import Request, RequestPriority, RequestStatus
from app.models.user import User, UserRole
from app.schemas.request import RequestUpdate


async def create_request(
    db: AsyncSession, title: str, description: str | None, priority: RequestPriority, author_id: int
) -> Request:
    req = Request(title=title, description=description, priority=priority, author_id=author_id)
    db.add(req)
    await db.commit()
    await db.refresh(req)
    return req


async def get_request(db: AsyncSession, request_id: int) -> Request | None:
    result = await db.execute(select(Request).where(Request.id == request_id))
    return result.scalar_one_or_none()


def _priority_weight_expr():
    """SQL expression for priority weight ordering."""
    return case(
        (Request.priority == RequestPriority.low, 0),
        (Request.priority == RequestPriority.normal, 1),
        (Request.priority == RequestPriority.high, 2),
    )


async def list_requests(
    db: AsyncSession,
    *,
    author_id: int | None = None,
    search: str | None = None,
    status_filter: RequestStatus | None = None,
    priority_filter: RequestPriority | None = None,
    sort_by: str = "created_at",
    sort_order: str = "desc",
    page: int = 1,
    per_page: int = 20,
) -> tuple[list[Request], int]:
    query: Select = select(Request)
    count_query = select(func.count()).select_from(Request)

    # Filters
    if author_id is not None:
        query = query.where(Request.author_id == author_id)
        count_query = count_query.where(Request.author_id == author_id)

    if search:
        search_pattern = f"%{search}%"
        search_filter = Request.title.ilike(search_pattern) | Request.description.ilike(search_pattern)
        query = query.where(search_filter)
        count_query = count_query.where(search_filter)

    if status_filter:
        query = query.where(Request.status == status_filter)
        count_query = count_query.where(Request.status == status_filter)

    if priority_filter:
        query = query.where(Request.priority == priority_filter)
        count_query = count_query.where(Request.priority == priority_filter)

    # Sorting
    if sort_by == "priority":
        order_col = _priority_weight_expr()
    else:
        order_col = Request.created_at

    if sort_order == "asc":
        query = query.order_by(order_col.asc())
    else:
        query = query.order_by(order_col.desc())

    # Count
    total_result = await db.execute(count_query)
    total = total_result.scalar() or 0

    # Pagination
    offset = (page - 1) * per_page
    query = query.offset(offset).limit(per_page)

    result = await db.execute(query)
    items = list(result.scalars().all())
    return items, total


def validate_request_update(request: Request, data: RequestUpdate, current_user: User) -> dict:
    """Validate and return only the fields the current user is allowed to update."""
    update_data = data.model_dump(exclude_unset=True)

    if not update_data:
        raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail="No fields to update")

    if current_user.role == UserRole.admin:
        # Admin can update everything
        return update_data

    if current_user.role == UserRole.manager:
        # Manager can only change status and assignee_id
        forbidden = set(update_data.keys()) - {"status", "assignee_id"}
        if forbidden:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Managers cannot edit: {', '.join(forbidden)}",
            )
        # Cannot revert done
        if "status" in update_data and request.status == RequestStatus.done:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST, detail="Cannot revert a done request"
            )
        return update_data

    # Role: user
    if request.author_id != current_user.id:
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Cannot edit another user's request")

    if request.status == RequestStatus.done:
        raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail="Cannot edit a done request")

    # User can edit title, description, priority, status of own non-done requests
    forbidden = set(update_data.keys()) - {"title", "description", "priority", "status"}
    if forbidden:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail=f"Users cannot edit: {', '.join(forbidden)}",
        )

    return update_data


async def update_request(db: AsyncSession, request: Request, data: RequestUpdate, current_user: User) -> Request:
    update_data = validate_request_update(request, data, current_user)

    # If assignee_id is set, validate it's a manager
    if "assignee_id" in update_data and update_data["assignee_id"] is not None:
        from app.services.user import get_user_by_id

        assignee = await get_user_by_id(db, update_data["assignee_id"])
        if not assignee or assignee.role not in (UserRole.manager, UserRole.admin):
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST, detail="Assignee must be a manager or admin"
            )

    for key, value in update_data.items():
        setattr(request, key, value)

    await db.commit()
    await db.refresh(request)
    return request


async def delete_request(db: AsyncSession, request: Request) -> None:
    await db.delete(request)
    await db.commit()
```

- [ ] **Step 2: Create `app/api/requests.py`**

```python
import math

from fastapi import APIRouter, Depends, HTTPException, Query, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.dependencies.auth import get_current_user, require_role
from app.dependencies.database import get_db
from app.models.request import RequestPriority, RequestStatus
from app.models.user import User, UserRole
from app.schemas.request import RequestCreate, RequestListResponse, RequestResponse, RequestUpdate
from app.services.request import create_request, delete_request, get_request, list_requests, update_request

router = APIRouter(prefix="/requests", tags=["requests"])


@router.get("", response_model=RequestListResponse)
async def get_requests(
    search: str | None = None,
    status_filter: RequestStatus | None = Query(None, alias="status"),
    priority_filter: RequestPriority | None = Query(None, alias="priority"),
    sort_by: str = Query("created_at", pattern="^(created_at|priority)$"),
    sort_order: str = Query("desc", pattern="^(asc|desc)$"),
    page: int = Query(1, ge=1),
    per_page: int = Query(20, ge=5, le=100),
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    # Users can only see their own requests
    author_id = current_user.id if current_user.role == UserRole.user else None

    items, total = await list_requests(
        db,
        author_id=author_id,
        search=search,
        status_filter=status_filter,
        priority_filter=priority_filter,
        sort_by=sort_by,
        sort_order=sort_order,
        page=page,
        per_page=per_page,
    )

    pages = math.ceil(total / per_page) if total > 0 else 0

    return RequestListResponse(
        items=[RequestResponse.model_validate(item) for item in items],
        total=total,
        page=page,
        per_page=per_page,
        pages=pages,
    )


@router.post("", response_model=RequestResponse, status_code=status.HTTP_201_CREATED)
async def create_new_request(
    data: RequestCreate,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    req = await create_request(db, data.title, data.description, data.priority, current_user.id)
    return RequestResponse.model_validate(req)


@router.get("/{request_id}", response_model=RequestResponse)
async def get_single_request(
    request_id: int,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    req = await get_request(db, request_id)
    if not req:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Request not found")

    # Users can only view their own
    if current_user.role == UserRole.user and req.author_id != current_user.id:
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Access denied")

    return RequestResponse.model_validate(req)


@router.patch("/{request_id}", response_model=RequestResponse)
async def update_existing_request(
    request_id: int,
    data: RequestUpdate,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    req = await get_request(db, request_id)
    if not req:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Request not found")

    updated = await update_request(db, req, data, current_user)
    return RequestResponse.model_validate(updated)


@router.delete("/{request_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_existing_request(
    request_id: int,
    _: User = Depends(require_role(UserRole.admin)),
    db: AsyncSession = Depends(get_db),
):
    req = await get_request(db, request_id)
    if not req:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Request not found")

    await delete_request(db, req)
```

- [ ] **Step 3: Update `app/api/__init__.py`**

```python
from fastapi import APIRouter

from app.api.auth import router as auth_router
from app.api.requests import router as requests_router
from app.api.users import router as users_router

api_router = APIRouter(prefix="/api")
api_router.include_router(auth_router)
api_router.include_router(users_router)
api_router.include_router(requests_router)
```

- [ ] **Step 4: Write request CRUD tests in `tests/test_requests.py`**

```python
import pytest
from httpx import AsyncClient

from app.models.user import UserRole
from tests.conftest import create_test_user


async def _login(client: AsyncClient, username: str, password: str) -> str:
    resp = await client.post("/api/auth/login", data={"username": username, "password": password})
    return resp.json()["access_token"]


@pytest.mark.asyncio
async def test_create_request(client: AsyncClient, db_session):
    await create_test_user(db_session, "creator", "password123")
    token = await _login(client, "creator", "password123")

    response = await client.post(
        "/api/requests",
        json={"title": "Fix the bug", "description": "It's broken", "priority": "high"},
        headers={"Authorization": f"Bearer {token}"},
    )
    assert response.status_code == 201
    data = response.json()
    assert data["title"] == "Fix the bug"
    assert data["status"] == "new"
    assert data["priority"] == "high"
    assert data["author"]["username"] == "creator"


@pytest.mark.asyncio
async def test_list_requests_user_sees_own(client: AsyncClient, db_session):
    user1 = await create_test_user(db_session, "user1", "password123")
    user2 = await create_test_user(db_session, "user2", "password123")

    token1 = await _login(client, "user1", "password123")
    token2 = await _login(client, "user2", "password123")

    await client.post("/api/requests", json={"title": "User1 request"}, headers={"Authorization": f"Bearer {token1}"})
    await client.post("/api/requests", json={"title": "User2 request"}, headers={"Authorization": f"Bearer {token2}"})

    resp = await client.get("/api/requests", headers={"Authorization": f"Bearer {token1}"})
    assert resp.status_code == 200
    items = resp.json()["items"]
    assert len(items) == 1
    assert items[0]["title"] == "User1 request"


@pytest.mark.asyncio
async def test_list_requests_manager_sees_all(client: AsyncClient, db_session):
    await create_test_user(db_session, "mgruser", "password123", UserRole.user)
    await create_test_user(db_session, "mgr", "password123", UserRole.manager)

    token_user = await _login(client, "mgruser", "password123")
    token_mgr = await _login(client, "mgr", "password123")

    await client.post("/api/requests", json={"title": "A request"}, headers={"Authorization": f"Bearer {token_user}"})

    resp = await client.get("/api/requests", headers={"Authorization": f"Bearer {token_mgr}"})
    assert resp.status_code == 200
    assert resp.json()["total"] >= 1


@pytest.mark.asyncio
async def test_search_requests(client: AsyncClient, db_session):
    await create_test_user(db_session, "searcher", "password123")
    token = await _login(client, "searcher", "password123")

    await client.post("/api/requests", json={"title": "Login page broken"}, headers={"Authorization": f"Bearer {token}"})
    await client.post("/api/requests", json={"title": "Dashboard update"}, headers={"Authorization": f"Bearer {token}"})

    resp = await client.get("/api/requests?search=login", headers={"Authorization": f"Bearer {token}"})
    assert resp.json()["total"] == 1
    assert "login" in resp.json()["items"][0]["title"].lower()


@pytest.mark.asyncio
async def test_pagination(client: AsyncClient, db_session):
    await create_test_user(db_session, "paguser", "password123")
    token = await _login(client, "paguser", "password123")

    for i in range(15):
        await client.post("/api/requests", json={"title": f"Request {i}"}, headers={"Authorization": f"Bearer {token}"})

    resp = await client.get("/api/requests?page=1&per_page=5", headers={"Authorization": f"Bearer {token}"})
    data = resp.json()
    assert len(data["items"]) == 5
    assert data["total"] == 15
    assert data["pages"] == 3


@pytest.mark.asyncio
async def test_delete_request_admin_only(client: AsyncClient, db_session):
    user = await create_test_user(db_session, "deluser", "password123")
    admin = await create_test_user(db_session, "deladmin", "password123", UserRole.admin)

    token_user = await _login(client, "deluser", "password123")
    token_admin = await _login(client, "deladmin", "password123")

    create_resp = await client.post(
        "/api/requests", json={"title": "To delete"}, headers={"Authorization": f"Bearer {token_user}"}
    )
    req_id = create_resp.json()["id"]

    # User cannot delete
    resp = await client.delete(f"/api/requests/{req_id}", headers={"Authorization": f"Bearer {token_user}"})
    assert resp.status_code == 403

    # Admin can delete
    resp = await client.delete(f"/api/requests/{req_id}", headers={"Authorization": f"Bearer {token_admin}"})
    assert resp.status_code == 204
```

- [ ] **Step 5: Write permission tests in `tests/test_permissions.py`**

```python
import pytest
from httpx import AsyncClient

from app.models.user import UserRole
from tests.conftest import create_test_user


async def _login(client: AsyncClient, username: str, password: str) -> str:
    resp = await client.post("/api/auth/login", data={"username": username, "password": password})
    return resp.json()["access_token"]


@pytest.mark.asyncio
async def test_user_cannot_edit_done_request(client: AsyncClient, db_session):
    user = await create_test_user(db_session, "doneuser", "password123")
    admin = await create_test_user(db_session, "doneadmin", "password123", UserRole.admin)

    token_user = await _login(client, "doneuser", "password123")
    token_admin = await _login(client, "doneadmin", "password123")

    # Create and mark as done
    create_resp = await client.post(
        "/api/requests", json={"title": "Done req"}, headers={"Authorization": f"Bearer {token_user}"}
    )
    req_id = create_resp.json()["id"]

    # Admin marks as done
    await client.patch(
        f"/api/requests/{req_id}", json={"status": "done"}, headers={"Authorization": f"Bearer {token_admin}"}
    )

    # User tries to edit — should fail
    resp = await client.patch(
        f"/api/requests/{req_id}", json={"title": "New title"}, headers={"Authorization": f"Bearer {token_user}"}
    )
    assert resp.status_code == 400


@pytest.mark.asyncio
async def test_manager_cannot_edit_title(client: AsyncClient, db_session):
    user = await create_test_user(db_session, "titleuser", "password123")
    mgr = await create_test_user(db_session, "titlemgr", "password123", UserRole.manager)

    token_user = await _login(client, "titleuser", "password123")
    token_mgr = await _login(client, "titlemgr", "password123")

    create_resp = await client.post(
        "/api/requests", json={"title": "Original title"}, headers={"Authorization": f"Bearer {token_user}"}
    )
    req_id = create_resp.json()["id"]

    resp = await client.patch(
        f"/api/requests/{req_id}", json={"title": "Changed by manager"}, headers={"Authorization": f"Bearer {token_mgr}"}
    )
    assert resp.status_code == 403


@pytest.mark.asyncio
async def test_manager_cannot_revert_done(client: AsyncClient, db_session):
    admin = await create_test_user(db_session, "revertadmin", "password123", UserRole.admin)
    mgr = await create_test_user(db_session, "revertmgr", "password123", UserRole.manager)

    token_admin = await _login(client, "revertadmin", "password123")
    token_mgr = await _login(client, "revertmgr", "password123")

    create_resp = await client.post(
        "/api/requests", json={"title": "Revert test"}, headers={"Authorization": f"Bearer {token_admin}"}
    )
    req_id = create_resp.json()["id"]

    await client.patch(
        f"/api/requests/{req_id}", json={"status": "done"}, headers={"Authorization": f"Bearer {token_admin}"}
    )

    resp = await client.patch(
        f"/api/requests/{req_id}", json={"status": "in_progress"}, headers={"Authorization": f"Bearer {token_mgr}"}
    )
    assert resp.status_code == 400


@pytest.mark.asyncio
async def test_admin_can_revert_done(client: AsyncClient, db_session):
    admin = await create_test_user(db_session, "adminrevert", "password123", UserRole.admin)
    token = await _login(client, "adminrevert", "password123")

    create_resp = await client.post(
        "/api/requests", json={"title": "Admin revert"}, headers={"Authorization": f"Bearer {token}"}
    )
    req_id = create_resp.json()["id"]

    await client.patch(f"/api/requests/{req_id}", json={"status": "done"}, headers={"Authorization": f"Bearer {token}"})

    resp = await client.patch(
        f"/api/requests/{req_id}", json={"status": "new"}, headers={"Authorization": f"Bearer {token}"}
    )
    assert resp.status_code == 200
    assert resp.json()["status"] == "new"


@pytest.mark.asyncio
async def test_manager_can_assign_to_manager(client: AsyncClient, db_session):
    user = await create_test_user(db_session, "assignuser", "password123")
    mgr1 = await create_test_user(db_session, "assignmgr1", "password123", UserRole.manager)
    mgr2 = await create_test_user(db_session, "assignmgr2", "password123", UserRole.manager)

    token_user = await _login(client, "assignuser", "password123")
    token_mgr = await _login(client, "assignmgr1", "password123")

    create_resp = await client.post(
        "/api/requests", json={"title": "Assign test"}, headers={"Authorization": f"Bearer {token_user}"}
    )
    req_id = create_resp.json()["id"]

    resp = await client.patch(
        f"/api/requests/{req_id}",
        json={"assignee_id": mgr2.id},
        headers={"Authorization": f"Bearer {token_mgr}"},
    )
    assert resp.status_code == 200
    assert resp.json()["assignee"]["username"] == "assignmgr2"


@pytest.mark.asyncio
async def test_cannot_assign_to_regular_user(client: AsyncClient, db_session):
    user = await create_test_user(db_session, "assignfail_user", "password123")
    mgr = await create_test_user(db_session, "assignfail_mgr", "password123", UserRole.manager)

    token_user = await _login(client, "assignfail_user", "password123")
    token_mgr = await _login(client, "assignfail_mgr", "password123")

    create_resp = await client.post(
        "/api/requests", json={"title": "Assign fail"}, headers={"Authorization": f"Bearer {token_user}"}
    )
    req_id = create_resp.json()["id"]

    resp = await client.patch(
        f"/api/requests/{req_id}",
        json={"assignee_id": user.id},
        headers={"Authorization": f"Bearer {token_mgr}"},
    )
    assert resp.status_code == 400


@pytest.mark.asyncio
async def test_user_cannot_see_other_request(client: AsyncClient, db_session):
    user1 = await create_test_user(db_session, "seeuser1", "password123")
    user2 = await create_test_user(db_session, "seeuser2", "password123")

    token1 = await _login(client, "seeuser1", "password123")
    token2 = await _login(client, "seeuser2", "password123")

    create_resp = await client.post(
        "/api/requests", json={"title": "Secret"}, headers={"Authorization": f"Bearer {token1}"}
    )
    req_id = create_resp.json()["id"]

    resp = await client.get(f"/api/requests/{req_id}", headers={"Authorization": f"Bearer {token2}"})
    assert resp.status_code == 403
```

- [ ] **Step 6: Run all tests**

```bash
cd backend
pytest -v
```

Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
cd backend
git add .
git commit -m "feat: request CRUD, server-side filters, pagination, and permission tests"
```

---

## Task 7: Frontend Project Scaffold + Shared Layer

**Files:**
- Create: Vite + React + TypeScript project in `frontend/`
- Create: `frontend/src/shared/api/client.ts`
- Create: `frontend/src/shared/api/types.ts`
- Create: `frontend/src/shared/lib/utils.ts`
- Create: `frontend/src/shared/hooks/use-debounce.ts`
- Create: `frontend/src/styles/globals.css`
- Modify: `frontend/package.json` (add deps)
- Modify: `frontend/vite.config.ts` (add proxy)

**Interfaces:**
- Consumes: nothing (first frontend task)
- Produces:
  - `apiClient` — Axios instance with interceptors (refresh logic)
  - `PaginatedResponse<T>`, `ApiError`, `User`, `RequestItem`, `RequestStatus`, `RequestPriority`, `UserRole` types
  - `cn()` — className merge utility
  - `useDebounce(value, delay)` — debounce hook

- [ ] **Step 1: Initialize Vite project**

```bash
cd frontend
npx -y create-vite@latest ./ --template react-ts
```

Note: If the directory already has files, Vite may prompt; answer yes to overwrite.

- [ ] **Step 2: Install dependencies**

```bash
cd frontend
npm install axios @tanstack/react-query zustand react-router-dom sonner
npm install -D tailwindcss@3 postcss autoprefixer @types/react-router-dom
npx tailwindcss init -p
```

- [ ] **Step 3: Configure Tailwind CSS**

Update `tailwind.config.js`:
```js
/** @type {import('tailwindcss').Config} */
export default {
  content: ["./index.html", "./src/**/*.{js,ts,jsx,tsx}"],
  theme: { extend: {} },
  plugins: [],
}
```

Update `src/styles/globals.css`:
```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

- [ ] **Step 4: Initialize shadcn/ui**

```bash
cd frontend
npx -y shadcn@latest init -d
```

Then install needed components:
```bash
npx shadcn@latest add button input label card dialog select badge skeleton toast dropdown-menu table separator
```

- [ ] **Step 5: Create `src/shared/api/types.ts`**

```typescript
export type UserRole = "user" | "manager" | "admin";

export interface User {
  id: number;
  username: string;
  role: UserRole;
  created_at: string;
}

export type RequestStatus = "new" | "in_progress" | "done";
export type RequestPriority = "low" | "normal" | "high";

export interface Author {
  id: number;
  username: string;
}

export interface RequestItem {
  id: number;
  title: string;
  description: string | null;
  status: RequestStatus;
  priority: RequestPriority;
  author: Author;
  assignee: Author | null;
  created_at: string;
  updated_at: string;
}

export interface PaginatedResponse<T> {
  items: T[];
  total: number;
  page: number;
  per_page: number;
  pages: number;
}

export interface ApiError {
  detail: string;
}

export interface TokenResponse {
  access_token: string;
  token_type: string;
  user?: User;
}
```

- [ ] **Step 6: Create `src/shared/api/client.ts`**

```typescript
import axios, { AxiosError, InternalAxiosRequestConfig } from "axios";

const API_URL = import.meta.env.VITE_API_URL || "/api";

export const apiClient = axios.create({
  baseURL: API_URL,
  withCredentials: true, // send cookies for refresh
});

let isRefreshing = false;
let failedQueue: Array<{
  resolve: (token: string) => void;
  reject: (err: unknown) => void;
}> = [];

const processQueue = (error: unknown, token: string | null) => {
  failedQueue.forEach((prom) => {
    if (token) prom.resolve(token);
    else prom.reject(error);
  });
  failedQueue = [];
};

// Will be set by auth store after initialization
let getAccessToken: () => string | null = () => null;
let setAccessToken: (token: string | null) => void = () => {};
let onLogout: () => void = () => {};

export const setAuthInterceptorCallbacks = (
  getter: () => string | null,
  setter: (token: string | null) => void,
  logout: () => void
) => {
  getAccessToken = getter;
  setAccessToken = setter;
  onLogout = logout;
};

// Request interceptor: attach access token
apiClient.interceptors.request.use((config: InternalAxiosRequestConfig) => {
  const token = getAccessToken();
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Response interceptor: handle 401 with refresh
apiClient.interceptors.response.use(
  (response) => response,
  async (error: AxiosError) => {
    const originalRequest = error.config;
    if (!originalRequest || error.response?.status !== 401) {
      return Promise.reject(error);
    }

    // Don't retry refresh/login endpoints
    if (
      originalRequest.url?.includes("/auth/refresh") ||
      originalRequest.url?.includes("/auth/login")
    ) {
      onLogout();
      return Promise.reject(error);
    }

    if (isRefreshing) {
      return new Promise((resolve, reject) => {
        failedQueue.push({
          resolve: (token: string) => {
            originalRequest.headers.Authorization = `Bearer ${token}`;
            resolve(apiClient(originalRequest));
          },
          reject,
        });
      });
    }

    isRefreshing = true;

    try {
      const { data } = await apiClient.post("/auth/refresh");
      const newToken = data.access_token;
      setAccessToken(newToken);
      processQueue(null, newToken);
      originalRequest.headers.Authorization = `Bearer ${newToken}`;
      return apiClient(originalRequest);
    } catch (refreshError) {
      processQueue(refreshError, null);
      onLogout();
      return Promise.reject(refreshError);
    } finally {
      isRefreshing = false;
    }
  }
);
```

- [ ] **Step 7: Create `src/shared/lib/utils.ts`**

```typescript
import { type ClassValue, clsx } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

- [ ] **Step 8: Create `src/shared/hooks/use-debounce.ts`**

```typescript
import { useEffect, useState } from "react";

export function useDebounce<T>(value: T, delay: number = 300): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}
```

- [ ] **Step 9: Configure Vite proxy for development**

Update `vite.config.ts`:
```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import path from "path";

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./src"),
    },
  },
  server: {
    proxy: {
      "/api": {
        target: "http://localhost:8000",
        changeOrigin: true,
      },
    },
  },
});
```

- [ ] **Step 10: Verify build compiles**

```bash
cd frontend
npx tsc --noEmit
npm run build
```

Expected: No errors

- [ ] **Step 11: Commit**

```bash
cd frontend
git add .
git commit -m "feat: project scaffold with Vite, Tailwind, shadcn/ui, Axios client, shared types"
```

---

## Task 8: Frontend Auth Feature (Store, API, Forms, Guards)

**Files:**
- Create: `frontend/src/features/auth/store/auth.store.ts`
- Create: `frontend/src/features/auth/api/auth.api.ts`
- Create: `frontend/src/features/auth/api/auth.queries.ts`
- Create: `frontend/src/features/auth/ui/LoginForm.tsx`
- Create: `frontend/src/features/auth/ui/RegisterForm.tsx`
- Create: `frontend/src/features/auth/guards/ProtectedRoute.tsx`
- Create: `frontend/src/features/auth/guards/GuestRoute.tsx`
- Create: `frontend/src/pages/LoginPage.tsx`
- Create: `frontend/src/pages/RegisterPage.tsx`
- Create: `frontend/src/router.tsx`
- Create: `frontend/src/App.tsx`
- Modify: `frontend/src/main.tsx`

**Interfaces:**
- Consumes: `apiClient`, `User`, `TokenResponse` from Task 7
- Produces:
  - `useAuthStore()` — Zustand store: `{ user, accessToken, setAuth, logout, isAuthenticated }`
  - `useLogin()`, `useRegister()`, `useLogout()` — React Query mutations
  - `<ProtectedRoute>`, `<GuestRoute>` — route guards
  - `<LoginForm>`, `<RegisterForm>` — form components
  - `<LoginPage>`, `<RegisterPage>` — page components
  - App router with all routes

- [ ] **Step 1: Create `src/features/auth/store/auth.store.ts`**

```typescript
import { create } from "zustand";
import { User } from "@/shared/api/types";
import { setAuthInterceptorCallbacks } from "@/shared/api/client";

interface AuthState {
  user: User | null;
  accessToken: string | null;
  setAuth: (user: User, token: string) => void;
  setAccessToken: (token: string | null) => void;
  logout: () => void;
  isAuthenticated: () => boolean;
}

export const useAuthStore = create<AuthState>((set, get) => ({
  user: null,
  accessToken: null,

  setAuth: (user, token) => set({ user, accessToken: token }),

  setAccessToken: (token) => set({ accessToken: token }),

  logout: () => {
    set({ user: null, accessToken: null });
    window.location.href = "/login";
  },

  isAuthenticated: () => get().user !== null && get().accessToken !== null,
}));

// Wire up Axios interceptors with store
setAuthInterceptorCallbacks(
  () => useAuthStore.getState().accessToken,
  (token) => useAuthStore.getState().setAccessToken(token),
  () => useAuthStore.getState().logout()
);
```

- [ ] **Step 2: Create `src/features/auth/api/auth.api.ts`**

```typescript
import { apiClient } from "@/shared/api/client";
import { TokenResponse, User } from "@/shared/api/types";

export async function loginApi(username: string, password: string): Promise<TokenResponse> {
  const formData = new URLSearchParams();
  formData.append("username", username);
  formData.append("password", password);

  const { data } = await apiClient.post<TokenResponse>("/auth/login", formData, {
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
  });
  return data;
}

export async function registerApi(username: string, password: string): Promise<User> {
  const { data } = await apiClient.post<User>("/auth/register", { username, password });
  return data;
}

export async function logoutApi(): Promise<void> {
  await apiClient.post("/auth/logout");
}

export async function refreshApi(): Promise<TokenResponse> {
  const { data } = await apiClient.post<TokenResponse>("/auth/refresh");
  return data;
}

export async function getMeApi(): Promise<User> {
  const { data } = await apiClient.get<User>("/users/me");
  return data;
}
```

- [ ] **Step 3: Create `src/features/auth/api/auth.queries.ts`**

```typescript
import { useMutation } from "@tanstack/react-query";
import { useNavigate } from "react-router-dom";
import { toast } from "sonner";
import { AxiosError } from "axios";

import { useAuthStore } from "../store/auth.store";
import { loginApi, registerApi, logoutApi } from "./auth.api";
import { ApiError } from "@/shared/api/types";

export function useLogin() {
  const setAuth = useAuthStore((s) => s.setAuth);
  const navigate = useNavigate();

  return useMutation({
    mutationFn: ({ username, password }: { username: string; password: string }) =>
      loginApi(username, password),
    onSuccess: (data) => {
      if (data.user) {
        setAuth(data.user, data.access_token);
        navigate("/");
      }
    },
    onError: (error: AxiosError<ApiError>) => {
      toast.error(error.response?.data?.detail || "Login failed");
    },
  });
}

export function useRegister() {
  const navigate = useNavigate();

  return useMutation({
    mutationFn: ({ username, password }: { username: string; password: string }) =>
      registerApi(username, password),
    onSuccess: () => {
      toast.success("Registration successful! Please log in.");
      navigate("/login");
    },
    onError: (error: AxiosError<ApiError>) => {
      toast.error(error.response?.data?.detail || "Registration failed");
    },
  });
}

export function useLogout() {
  const logout = useAuthStore((s) => s.logout);

  return useMutation({
    mutationFn: logoutApi,
    onSuccess: () => logout(),
    onError: () => logout(), // logout anyway
  });
}
```

- [ ] **Step 4: Create `src/features/auth/ui/LoginForm.tsx`**

```tsx
import { useState } from "react";
import { Link } from "react-router-dom";
import { Button } from "@/shared/ui/button";
import { Input } from "@/shared/ui/input";
import { Label } from "@/shared/ui/label";
import { Card, CardContent, CardDescription, CardFooter, CardHeader, CardTitle } from "@/shared/ui/card";
import { useLogin } from "../api/auth.queries";

export function LoginForm() {
  const [username, setUsername] = useState("");
  const [password, setPassword] = useState("");
  const login = useLogin();

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    login.mutate({ username, password });
  };

  return (
    <Card className="w-full max-w-md">
      <CardHeader>
        <CardTitle className="text-2xl">Login</CardTitle>
        <CardDescription>Enter your credentials to access the system</CardDescription>
      </CardHeader>
      <form onSubmit={handleSubmit}>
        <CardContent className="space-y-4">
          <div className="space-y-2">
            <Label htmlFor="username">Username</Label>
            <Input id="username" value={username} onChange={(e) => setUsername(e.target.value)} required />
          </div>
          <div className="space-y-2">
            <Label htmlFor="password">Password</Label>
            <Input id="password" type="password" value={password} onChange={(e) => setPassword(e.target.value)} required />
          </div>
        </CardContent>
        <CardFooter className="flex flex-col gap-4">
          <Button type="submit" className="w-full" disabled={login.isPending}>
            {login.isPending ? "Logging in..." : "Login"}
          </Button>
          <p className="text-sm text-muted-foreground">
            Don't have an account?{" "}
            <Link to="/register" className="text-primary hover:underline">Register</Link>
          </p>
        </CardFooter>
      </form>
    </Card>
  );
}
```

- [ ] **Step 5: Create `src/features/auth/ui/RegisterForm.tsx`**

```tsx
import { useState } from "react";
import { Link } from "react-router-dom";
import { Button } from "@/shared/ui/button";
import { Input } from "@/shared/ui/input";
import { Label } from "@/shared/ui/label";
import { Card, CardContent, CardDescription, CardFooter, CardHeader, CardTitle } from "@/shared/ui/card";
import { useRegister } from "../api/auth.queries";

export function RegisterForm() {
  const [username, setUsername] = useState("");
  const [password, setPassword] = useState("");
  const [confirmPassword, setConfirmPassword] = useState("");
  const register = useRegister();

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (password !== confirmPassword) return;
    register.mutate({ username, password });
  };

  return (
    <Card className="w-full max-w-md">
      <CardHeader>
        <CardTitle className="text-2xl">Register</CardTitle>
        <CardDescription>Create a new account</CardDescription>
      </CardHeader>
      <form onSubmit={handleSubmit}>
        <CardContent className="space-y-4">
          <div className="space-y-2">
            <Label htmlFor="reg-username">Username</Label>
            <Input id="reg-username" value={username} onChange={(e) => setUsername(e.target.value)} minLength={3} maxLength={50} required />
          </div>
          <div className="space-y-2">
            <Label htmlFor="reg-password">Password</Label>
            <Input id="reg-password" type="password" value={password} onChange={(e) => setPassword(e.target.value)} minLength={6} required />
          </div>
          <div className="space-y-2">
            <Label htmlFor="reg-confirm">Confirm Password</Label>
            <Input
              id="reg-confirm"
              type="password"
              value={confirmPassword}
              onChange={(e) => setConfirmPassword(e.target.value)}
              minLength={6}
              required
            />
            {confirmPassword && password !== confirmPassword && (
              <p className="text-sm text-destructive">Passwords don't match</p>
            )}
          </div>
        </CardContent>
        <CardFooter className="flex flex-col gap-4">
          <Button type="submit" className="w-full" disabled={register.isPending || password !== confirmPassword}>
            {register.isPending ? "Creating account..." : "Register"}
          </Button>
          <p className="text-sm text-muted-foreground">
            Already have an account?{" "}
            <Link to="/login" className="text-primary hover:underline">Login</Link>
          </p>
        </CardFooter>
      </form>
    </Card>
  );
}
```

- [ ] **Step 6: Create guards, pages, router, and App**

`src/features/auth/guards/ProtectedRoute.tsx`:
```tsx
import { Navigate, Outlet } from "react-router-dom";
import { useAuthStore } from "../store/auth.store";

export function ProtectedRoute() {
  const isAuthenticated = useAuthStore((s) => s.isAuthenticated());
  return isAuthenticated ? <Outlet /> : <Navigate to="/login" replace />;
}
```

`src/features/auth/guards/GuestRoute.tsx`:
```tsx
import { Navigate, Outlet } from "react-router-dom";
import { useAuthStore } from "../store/auth.store";

export function GuestRoute() {
  const isAuthenticated = useAuthStore((s) => s.isAuthenticated());
  return isAuthenticated ? <Navigate to="/" replace /> : <Outlet />;
}
```

`src/pages/LoginPage.tsx`:
```tsx
import { LoginForm } from "@/features/auth/ui/LoginForm";

export function LoginPage() {
  return (
    <div className="flex min-h-screen items-center justify-center bg-background p-4">
      <LoginForm />
    </div>
  );
}
```

`src/pages/RegisterPage.tsx`:
```tsx
import { RegisterForm } from "@/features/auth/ui/RegisterForm";

export function RegisterPage() {
  return (
    <div className="flex min-h-screen items-center justify-center bg-background p-4">
      <RegisterForm />
    </div>
  );
}
```

`src/pages/DashboardPage.tsx` (placeholder — will be built in Task 9):
```tsx
export function DashboardPage() {
  return <div className="p-8"><h1 className="text-2xl font-bold">Dashboard</h1><p>Coming soon...</p></div>;
}
```

`src/router.tsx`:
```tsx
import { createBrowserRouter } from "react-router-dom";
import { ProtectedRoute } from "@/features/auth/guards/ProtectedRoute";
import { GuestRoute } from "@/features/auth/guards/GuestRoute";
import { LoginPage } from "@/pages/LoginPage";
import { RegisterPage } from "@/pages/RegisterPage";
import { DashboardPage } from "@/pages/DashboardPage";

export const router = createBrowserRouter([
  {
    element: <GuestRoute />,
    children: [
      { path: "/login", element: <LoginPage /> },
      { path: "/register", element: <RegisterPage /> },
    ],
  },
  {
    element: <ProtectedRoute />,
    children: [
      { path: "/", element: <DashboardPage /> },
    ],
  },
]);
```

`src/App.tsx`:
```tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { RouterProvider } from "react-router-dom";
import { Toaster } from "sonner";
import { router } from "./router";

const queryClient = new QueryClient({
  defaultOptions: {
    queries: { retry: 1, refetchOnWindowFocus: false },
  },
});

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <RouterProvider router={router} />
      <Toaster richColors position="top-right" />
    </QueryClientProvider>
  );
}
```

`src/main.tsx`:
```tsx
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";
import "./styles/globals.css";

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

- [ ] **Step 7: Verify build**

```bash
cd frontend
npx tsc --noEmit
npm run build
```

Expected: No errors

- [ ] **Step 8: Commit**

```bash
cd frontend
git add .
git commit -m "feat: auth feature — store, API, forms, guards, routing"
```

---

## Task 9: Frontend Dashboard — Request List, Filters, CRUD

**Files:**
- Create: `frontend/src/features/requests/types.ts`
- Create: `frontend/src/features/requests/api/requests.api.ts`
- Create: `frontend/src/features/requests/api/requests.queries.ts`
- Create: `frontend/src/features/requests/ui/RequestStatusBadge.tsx`
- Create: `frontend/src/features/requests/ui/RequestCard.tsx`
- Create: `frontend/src/features/requests/ui/RequestFilters.tsx`
- Create: `frontend/src/features/requests/ui/RequestForm.tsx`
- Create: `frontend/src/features/requests/ui/RequestList.tsx`
- Modify: `frontend/src/pages/DashboardPage.tsx` (replace placeholder)
- Modify: `frontend/src/router.tsx` (add layout with nav)

**Interfaces:**
- Consumes: `apiClient`, `RequestItem`, `PaginatedResponse`, `useAuthStore`, shadcn/ui components
- Produces:
  - `useRequests(filters)` — paginated request list query
  - `useCreateRequest()`, `useUpdateRequest()`, `useDeleteRequest()` — mutations
  - Full dashboard UI with filters, card list, create/edit dialog, pagination

- [ ] **Step 1: Create `src/features/requests/types.ts`**

```typescript
import { RequestPriority, RequestStatus } from "@/shared/api/types";

export interface RequestFilters {
  search?: string;
  status?: RequestStatus;
  priority?: RequestPriority;
  sort_by?: "created_at" | "priority";
  sort_order?: "asc" | "desc";
  page?: number;
  per_page?: number;
}
```

- [ ] **Step 2: Create `src/features/requests/api/requests.api.ts`**

```typescript
import { apiClient } from "@/shared/api/client";
import { PaginatedResponse, RequestItem } from "@/shared/api/types";
import { RequestFilters } from "../types";

export async function fetchRequests(filters: RequestFilters): Promise<PaginatedResponse<RequestItem>> {
  const params = new URLSearchParams();
  if (filters.search) params.append("search", filters.search);
  if (filters.status) params.append("status", filters.status);
  if (filters.priority) params.append("priority", filters.priority);
  if (filters.sort_by) params.append("sort_by", filters.sort_by);
  if (filters.sort_order) params.append("sort_order", filters.sort_order);
  params.append("page", String(filters.page || 1));
  params.append("per_page", String(filters.per_page || 20));

  const { data } = await apiClient.get<PaginatedResponse<RequestItem>>(`/requests?${params}`);
  return data;
}

export async function createRequest(payload: {
  title: string;
  description?: string;
  priority?: string;
}): Promise<RequestItem> {
  const { data } = await apiClient.post<RequestItem>("/requests", payload);
  return data;
}

export async function updateRequest(
  id: number,
  payload: Record<string, unknown>
): Promise<RequestItem> {
  const { data } = await apiClient.patch<RequestItem>(`/requests/${id}`, payload);
  return data;
}

export async function deleteRequest(id: number): Promise<void> {
  await apiClient.delete(`/requests/${id}`);
}
```

- [ ] **Step 3: Create `src/features/requests/api/requests.queries.ts`**

```typescript
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { toast } from "sonner";
import { AxiosError } from "axios";

import { ApiError } from "@/shared/api/types";
import { RequestFilters } from "../types";
import { createRequest, deleteRequest, fetchRequests, updateRequest } from "./requests.api";

export function useRequests(filters: RequestFilters) {
  return useQuery({
    queryKey: ["requests", filters],
    queryFn: () => fetchRequests(filters),
  });
}

export function useCreateRequest() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: createRequest,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["requests"] });
      toast.success("Request created");
    },
    onError: (error: AxiosError<ApiError>) => {
      toast.error(error.response?.data?.detail || "Failed to create request");
    },
  });
}

export function useUpdateRequest() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: ({ id, ...payload }: { id: number } & Record<string, unknown>) =>
      updateRequest(id, payload),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["requests"] });
      toast.success("Request updated");
    },
    onError: (error: AxiosError<ApiError>) => {
      toast.error(error.response?.data?.detail || "Failed to update request");
    },
  });
}

export function useDeleteRequest() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: deleteRequest,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["requests"] });
      toast.success("Request deleted");
    },
    onError: (error: AxiosError<ApiError>) => {
      toast.error(error.response?.data?.detail || "Failed to delete request");
    },
  });
}
```

- [ ] **Step 4: Create UI components (RequestStatusBadge, RequestCard, RequestFilters, RequestForm, RequestList)**

These are substantial UI components. Create each file with the full implementation:

`src/features/requests/ui/RequestStatusBadge.tsx` — Badge with color-coded status.
`src/features/requests/ui/RequestCard.tsx` — Card displaying request info with click-to-edit.
`src/features/requests/ui/RequestFilters.tsx` — Toolbar with search input, status/priority dropdowns, sort controls.
`src/features/requests/ui/RequestForm.tsx` — Dialog form for create/edit, adapts fields to user role.
`src/features/requests/ui/RequestList.tsx` — Grid of RequestCards with pagination buttons.

Each component uses shadcn/ui primitives, reads user role from `useAuthStore`, and calls the appropriate query hooks.

*Full code for each component is provided in the implementation — each file should be 30-80 lines max, focused on one responsibility.*

- [ ] **Step 5: Update `src/pages/DashboardPage.tsx`**

Replace the placeholder with the full dashboard layout:

```tsx
import { useState } from "react";
import { RequestList } from "@/features/requests/ui/RequestList";
import { RequestFilters } from "@/features/requests/ui/RequestFilters";
import { RequestForm } from "@/features/requests/ui/RequestForm";
import { Button } from "@/shared/ui/button";
import { useAuthStore } from "@/features/auth/store/auth.store";
import { useLogout } from "@/features/auth/api/auth.queries";
import { RequestFilters as Filters } from "@/features/requests/types";
import { Plus } from "lucide-react";

export function DashboardPage() {
  const user = useAuthStore((s) => s.user);
  const logout = useLogout();
  const [filters, setFilters] = useState<Filters>({ page: 1, per_page: 20 });
  const [createOpen, setCreateOpen] = useState(false);

  return (
    <div className="min-h-screen bg-background">
      <header className="border-b bg-card px-6 py-4 flex items-center justify-between">
        <h1 className="text-xl font-bold">Issue Tracker</h1>
        <div className="flex items-center gap-4">
          <span className="text-sm text-muted-foreground">
            {user?.username} ({user?.role})
          </span>
          {user?.role === "admin" && (
            <Button variant="outline" size="sm" asChild>
              <a href="/admin">Admin</a>
            </Button>
          )}
          <Button variant="ghost" size="sm" onClick={() => logout.mutate()}>
            Logout
          </Button>
        </div>
      </header>
      <main className="max-w-6xl mx-auto p-6 space-y-6">
        <div className="flex items-center justify-between">
          <RequestFilters filters={filters} onChange={setFilters} />
          <Button onClick={() => setCreateOpen(true)}>
            <Plus className="w-4 h-4 mr-2" />
            Create Request
          </Button>
        </div>
        <RequestList filters={filters} onPageChange={(page) => setFilters((f) => ({ ...f, page }))} />
        <RequestForm open={createOpen} onClose={() => setCreateOpen(false)} />
      </main>
    </div>
  );
}
```

- [ ] **Step 6: Verify build**

```bash
cd frontend
npx tsc --noEmit
npm run build
```

Expected: No errors

- [ ] **Step 7: Commit**

```bash
cd frontend
git add .
git commit -m "feat: dashboard with request list, filters, CRUD, and pagination"
```

---

## Task 10: Frontend Admin Panel

**Files:**
- Create: `frontend/src/features/admin/api/users.api.ts`
- Create: `frontend/src/features/admin/api/users.queries.ts`
- Create: `frontend/src/features/admin/ui/UserList.tsx`
- Create: `frontend/src/features/admin/ui/RoleSelect.tsx`
- Create: `frontend/src/pages/AdminPage.tsx`
- Modify: `frontend/src/router.tsx` (add admin route)

**Interfaces:**
- Consumes: `apiClient`, `User`, `UserRole`, `useAuthStore`
- Produces:
  - `useUsers()` — list all users query
  - `useUpdateUserRole()` — mutation
  - `<AdminPage>` — user management table with role dropdowns

- [ ] **Step 1: Create admin API and queries**

`src/features/admin/api/users.api.ts`:
```typescript
import { apiClient } from "@/shared/api/client";
import { User, UserRole } from "@/shared/api/types";

export async function fetchUsers(): Promise<User[]> {
  const { data } = await apiClient.get<User[]>("/users");
  return data;
}

export async function updateUserRole(userId: number, role: UserRole): Promise<User> {
  const { data } = await apiClient.patch<User>(`/users/${userId}/role`, { role });
  return data;
}
```

`src/features/admin/api/users.queries.ts`:
```typescript
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { toast } from "sonner";
import { AxiosError } from "axios";

import { ApiError, UserRole } from "@/shared/api/types";
import { fetchUsers, updateUserRole } from "./users.api";

export function useUsers() {
  return useQuery({ queryKey: ["users"], queryFn: fetchUsers });
}

export function useUpdateUserRole() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: ({ userId, role }: { userId: number; role: UserRole }) => updateUserRole(userId, role),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["users"] });
      toast.success("Role updated");
    },
    onError: (error: AxiosError<ApiError>) => {
      toast.error(error.response?.data?.detail || "Failed to update role");
    },
  });
}
```

- [ ] **Step 2: Create admin UI components and page**

`src/features/admin/ui/RoleSelect.tsx`:
```tsx
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/shared/ui/select";
import { UserRole } from "@/shared/api/types";

interface Props {
  value: UserRole;
  onChange: (role: UserRole) => void;
  disabled?: boolean;
}

export function RoleSelect({ value, onChange, disabled }: Props) {
  return (
    <Select value={value} onValueChange={(v) => onChange(v as UserRole)} disabled={disabled}>
      <SelectTrigger className="w-32">
        <SelectValue />
      </SelectTrigger>
      <SelectContent>
        <SelectItem value="user">User</SelectItem>
        <SelectItem value="manager">Manager</SelectItem>
        <SelectItem value="admin">Admin</SelectItem>
      </SelectContent>
    </Select>
  );
}
```

`src/features/admin/ui/UserList.tsx`:
```tsx
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from "@/shared/ui/table";
import { Skeleton } from "@/shared/ui/skeleton";
import { useAuthStore } from "@/features/auth/store/auth.store";
import { useUsers, useUpdateUserRole } from "../api/users.queries";
import { RoleSelect } from "./RoleSelect";

export function UserList() {
  const currentUser = useAuthStore((s) => s.user);
  const { data: users, isLoading } = useUsers();
  const updateRole = useUpdateUserRole();

  if (isLoading) {
    return <div className="space-y-2">{Array.from({ length: 5 }).map((_, i) => <Skeleton key={i} className="h-12 w-full" />)}</div>;
  }

  return (
    <Table>
      <TableHeader>
        <TableRow>
          <TableHead>ID</TableHead>
          <TableHead>Username</TableHead>
          <TableHead>Role</TableHead>
          <TableHead>Created</TableHead>
        </TableRow>
      </TableHeader>
      <TableBody>
        {users?.map((user) => (
          <TableRow key={user.id}>
            <TableCell>{user.id}</TableCell>
            <TableCell className="font-medium">{user.username}</TableCell>
            <TableCell>
              <RoleSelect
                value={user.role}
                onChange={(role) => updateRole.mutate({ userId: user.id, role })}
                disabled={user.id === currentUser?.id}
              />
            </TableCell>
            <TableCell className="text-muted-foreground">
              {new Date(user.created_at).toLocaleDateString()}
            </TableCell>
          </TableRow>
        ))}
      </TableBody>
    </Table>
  );
}
```

`src/pages/AdminPage.tsx`:
```tsx
import { Button } from "@/shared/ui/button";
import { UserList } from "@/features/admin/ui/UserList";
import { ArrowLeft } from "lucide-react";
import { Link } from "react-router-dom";

export function AdminPage() {
  return (
    <div className="min-h-screen bg-background">
      <header className="border-b bg-card px-6 py-4 flex items-center gap-4">
        <Link to="/"><Button variant="ghost" size="icon"><ArrowLeft className="w-4 h-4" /></Button></Link>
        <h1 className="text-xl font-bold">Admin — User Management</h1>
      </header>
      <main className="max-w-4xl mx-auto p-6">
        <UserList />
      </main>
    </div>
  );
}
```

- [ ] **Step 3: Update router to include admin route with role check**

Add to `src/router.tsx`:
```tsx
import { AdminPage } from "@/pages/AdminPage";

// Inside ProtectedRoute children:
{ path: "/admin", element: <AdminPage /> },
```

- [ ] **Step 4: Verify build**

```bash
cd frontend
npx tsc --noEmit
npm run build
```

Expected: No errors

- [ ] **Step 5: Commit**

```bash
cd frontend
git add .
git commit -m "feat: admin panel with user list and role management"
```

---

## Task 11: Docker Infrastructure

**Files:**
- Create: `backend/Dockerfile`
- Create: `backend/Dockerfile.dev`
- Create: `frontend/Dockerfile`
- Create: `frontend/Dockerfile.dev`
- Create: `frontend/nginx.conf`
- Create: `docker-compose.yml`
- Create: `docker-compose.dev.yml`
- Create: `.env.example`

**Interfaces:**
- Consumes: backend app at port 8000, frontend build output
- Produces: Full Docker Compose stack for both dev and production

- [ ] **Step 1: Create backend Dockerfiles**

`backend/Dockerfile`:
```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY pyproject.toml ./
RUN pip install --no-cache-dir .

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin
COPY . .
RUN mkdir -p data
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

`backend/Dockerfile.dev`:
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY pyproject.toml ./
RUN pip install -e ".[dev]"
COPY . .
```

- [ ] **Step 2: Create frontend Dockerfiles and nginx.conf**

`frontend/Dockerfile`:
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

`frontend/Dockerfile.dev`:
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
EXPOSE 5173
```

`frontend/nginx.conf`:
```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://backend:8000/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

- [ ] **Step 3: Create docker-compose files and .env.example**

`docker-compose.yml`:
```yaml
version: "3.8"
services:
  backend:
    build: ./backend
    ports: ["8000:8000"]
    volumes: ["./data:/app/data"]
    environment:
      - DATABASE_URL=sqlite+aiosqlite:///./data/app.db
      - JWT_SECRET=${JWT_SECRET:-change-me-in-production}
      - CORS_ORIGINS=http://localhost:3000
    restart: unless-stopped

  frontend:
    build: ./frontend
    ports: ["3000:80"]
    depends_on: [backend]
    restart: unless-stopped
```

`docker-compose.dev.yml`:
```yaml
version: "3.8"
services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
    ports: ["8000:8000"]
    volumes:
      - ./backend:/app
      - ./data:/app/data
    environment:
      - DATABASE_URL=sqlite+aiosqlite:///./data/app.db
      - JWT_SECRET=dev-secret-key
      - CORS_ORIGINS=http://localhost:5173
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    ports: ["5173:5173"]
    volumes:
      - ./frontend:/app
      - /app/node_modules
    command: npm run dev -- --host
    depends_on: [backend]
```

`.env.example`:
```
JWT_SECRET=your-secret-key-here
ADMIN_USERNAME=admin
ADMIN_PASSWORD=admin
```

- [ ] **Step 4: Test Docker Compose builds**

```bash
docker compose build
```

Expected: Both services build successfully

- [ ] **Step 5: Commit all repos**

```bash
cd backend && git add . && git commit -m "feat: Docker files for dev and production"
cd ../frontend && git add . && git commit -m "feat: Docker files and nginx config"
cd .. && git add . && git commit -m "feat: Docker Compose for dev and production"
```

---

## Task 12: CI/CD — GitHub Actions

**Files:**
- Create: `backend/.github/workflows/ci.yml`
- Create: `frontend/.github/workflows/ci.yml`

**Interfaces:**
- Consumes: test and lint commands from both repos
- Produces: CI pipelines that run on push/PR

- [ ] **Step 1: Create backend CI**

`backend/.github/workflows/ci.yml`:
```yaml
name: Backend CI
on: [push, pull_request]

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - name: Install dependencies
        run: pip install -e ".[dev]"
      - name: Lint
        run: ruff check .
      - name: Format check
        run: ruff format --check .
      - name: Test
        run: pytest -v
        env:
          JWT_SECRET: test-secret
```

- [ ] **Step 2: Create frontend CI**

`frontend/.github/workflows/ci.yml`:
```yaml
name: Frontend CI
on: [push, pull_request]

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
      - name: Install dependencies
        run: npm ci
      - name: Lint
        run: npm run lint
      - name: Type check
        run: npx tsc --noEmit
      - name: Test
        run: npm run test -- --run
```

- [ ] **Step 3: Commit**

```bash
cd backend && git add . && git commit -m "ci: GitHub Actions for lint and test"
cd ../frontend && git add . && git commit -m "ci: GitHub Actions for lint, typecheck, and test"
```

---

## Verification Plan

### Automated Tests

```bash
# Backend
cd backend && pytest -v

# Frontend
cd frontend && npm run test -- --run
```

### Manual Verification

1. Start dev environment: `docker compose -f docker-compose.dev.yml up --build`
2. Open `http://localhost:5173`
3. Register a new user → Login → Create requests → Filter/search/paginate
4. Login as admin (`admin:admin`) → See all requests → Delete a request → Go to Admin panel → Change user role to manager
5. Login as manager → See all requests → Change status, assign to self → Verify cannot edit title
6. Test token refresh: wait 15 min or manually expire token → verify auto-refresh
7. Production build: `docker compose up --build` → verify at `http://localhost:3000`
