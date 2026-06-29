# Design Specification: Issue Tracker (Fullstack FastAPI + React)

## 1. Overview

A lightweight internal helpdesk system for tracking requests. Employees create requests; managers and admins process them. The system enforces strict role-based access control and server-side data processing.

### Repository Structure

Three Git repositories in the workspace:

- `tz-github/` — Infrastructure repo: `docker-compose.yml`, shared docs, CI orchestration
- `tz-github/backend/` — Standalone Git repo for the FastAPI backend
- `tz-github/frontend/` — Standalone Git repo for the React frontend

## 2. Tech Stack

### Backend
- **Runtime:** Python 3.12
- **Framework:** FastAPI (async)
- **Database:** SQLite via `aiosqlite`
- **ORM:** SQLAlchemy 2.0 (async style)
- **Migrations:** Alembic (autogenerate)
- **Auth:** JWT (`python-jose[cryptography]`), bcrypt (`passlib[bcrypt]`)
- **Validation:** Pydantic v2
- **Linting:** Ruff (lint + format)
- **Testing:** Pytest + httpx (async test client)

### Frontend
- **Runtime:** Node.js 20
- **Framework:** React 18 + TypeScript
- **Build:** Vite
- **Styling:** Tailwind CSS v3
- **Components:** shadcn/ui
- **Data Fetching:** TanStack Query v5 (React Query)
- **Auth State:** Zustand (in-memory, no persist)
- **HTTP Client:** Axios (with interceptors)
- **Routing:** React Router v6
- **Linting:** ESLint + TypeScript strict
- **Testing:** Vitest + React Testing Library

### Infrastructure
- **Containers:** Docker (multi-stage builds)
- **Orchestration:** Docker Compose (production + development)
- **CI/CD:** GitHub Actions (per-repo)

## 3. Data Models

### User

| Field | Type | Constraints |
|-------|------|-------------|
| `id` | Integer | PK, auto-increment |
| `username` | String(50) | Unique, not null, 3-50 chars |
| `password_hash` | String(128) | Not null |
| `role` | Enum(`user`, `manager`, `admin`) | Default: `user` |
| `created_at` | DateTime | UTC, server default |

### Request

| Field | Type | Constraints |
|-------|------|-------------|
| `id` | Integer | PK, auto-increment |
| `title` | String(120) | Not null, 3-120 chars |
| `description` | String(1000) | Optional |
| `status` | Enum(`new`, `in_progress`, `done`) | Default: `new` |
| `priority` | Enum(`low`, `normal`, `high`) | Default: `normal` |
| `author_id` | Integer | FK → `User.id`, not null |
| `assignee_id` | Integer | FK → `User.id`, nullable |
| `created_at` | DateTime | UTC, server default |
| `updated_at` | DateTime | UTC, server default, auto-update |

**Priority weights for DB sorting:** `low=0`, `normal=1`, `high=2`.

### Indexes

- `User.username` — unique index
- `Request.author_id` — index (filter by author)
- `Request.status` — index (filter by status)
- `Request.priority` — index (sort by priority weight)
- `Request.created_at` — index (sort by date)

## 4. Roles & Permissions

### Guest (Unauthenticated)
- No access. Redirected to `/login`.

### User
- Registers with role `user` (hardcoded, cannot choose role).
- Views **only their own** requests.
- Creates new requests.
- Edits title, description, priority, and status of their own requests **unless** status is `done`.
- **Cannot** delete requests.

### Manager
- Views **all** requests.
- Changes the status of any request (except reverting `done`).
- Assigns requests to any user with role `manager` (including themselves).
- **Cannot** edit title/description (preserves author's intent).
- **Cannot** delete requests.
- **Cannot** revert `done` → other statuses.

### Admin
- Default account `admin:admin` created via DB seeding.
- Views, edits, and deletes **any** request.
- **Can** revert `done` → active statuses (bypasses business logic).
- **Can** change roles of other users (via API/UI).
- **Cannot** change their own role or delete their own account (protection).

### Permission Matrix

| Action | User | Manager | Admin |
|--------|------|---------|-------|
| View own requests | ✅ | ✅ | ✅ |
| View all requests | ❌ | ✅ | ✅ |
| Create request | ✅ | ✅ | ✅ |
| Edit own request text | ✅ (not done) | ❌ | ✅ |
| Edit request status | ✅ (own, not done) | ✅ (not revert done) | ✅ |
| Assign request | ❌ | ✅ (to managers) | ✅ |
| Delete request | ❌ | ❌ | ✅ |
| Revert done status | ❌ | ❌ | ✅ |
| Manage user roles | ❌ | ❌ | ✅ |
| View users list | ❌ | ❌ | ✅ |

## 5. Authentication & Security

### JWT Strategy: Access + Refresh Token Pair

| Token | Lifetime | Storage | Transport |
|-------|----------|---------|-----------|
| Access | 15 minutes | Frontend memory (Zustand) | `Authorization: Bearer <token>` header |
| Refresh | 7 days | httpOnly cookie | Cookie (automatic) |

### Auth Flow

1. **Login:** `POST /api/auth/login` → Returns `{ access_token, user }` + sets `refresh_token` httpOnly cookie.
2. **API Requests:** Axios interceptor attaches `Authorization: Bearer <accessToken>`.
3. **Token Refresh:** On 401 → Axios interceptor calls `POST /api/auth/refresh` (cookie sent automatically) → gets new access token.
4. **Concurrent Refresh:** If multiple requests get 401 simultaneously, a shared Promise prevents multiple refresh calls (queue pattern).
5. **Refresh Failure:** If refresh returns 401 → clear Zustand state → redirect to `/login`.
6. **Logout:** `POST /api/auth/logout` → clears refresh cookie + Zustand state.

### Password Security
- Hashing: bcrypt via `passlib[bcrypt]`
- Minimum password length: 6 characters
- Username: 3-50 characters, alphanumeric + underscores

### JWT Payload
```json
{
  "sub": 1,           // user.id
  "role": "user",     // user.role
  "exp": 1234567890   // expiration timestamp
}
```

## 6. API Specification

### Base URL: `/api`

### Auth Endpoints

| Method | Path | Auth | Request Body | Response |
|--------|------|------|-------------|----------|
| POST | `/auth/register` | No | `{ username, password }` | `201: { id, username, role }` |
| POST | `/auth/login` | No | `OAuth2 form: username, password` | `200: { access_token, token_type, user }` + Set-Cookie |
| POST | `/auth/refresh` | Cookie | — | `200: { access_token, token_type }` |
| POST | `/auth/logout` | Yes | — | `200: { detail: "logged out" }` + Clear Cookie |

### User Endpoints

| Method | Path | Auth | Request Body | Response |
|--------|------|------|-------------|----------|
| GET | `/users/me` | Yes | — | `200: { id, username, role, created_at }` |
| GET | `/users` | Admin | — | `200: [ { id, username, role, created_at } ]` |
| PATCH | `/users/{id}/role` | Admin | `{ role: "manager" }` | `200: { id, username, role }` |

### Request Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/requests` | Yes | List requests (filtered by role visibility) |
| POST | `/requests` | Yes | Create request |
| GET | `/requests/{id}` | Yes | Get single request |
| PATCH | `/requests/{id}` | Yes | Update request (fields depend on role) |
| DELETE | `/requests/{id}` | Admin | Delete request |

### GET `/requests` Query Parameters

| Param | Type | Description |
|-------|------|-------------|
| `search` | string | Full-text search in title and description (LIKE) |
| `status` | enum | Filter by status |
| `priority` | enum | Filter by priority |
| `sort_by` | enum(`created_at`, `priority`) | Sort field. Default: `created_at` |
| `sort_order` | enum(`asc`, `desc`) | Sort direction. Default: `desc` |
| `page` | int | Page number (1-indexed). Default: 1 |
| `per_page` | int | Items per page (5-100). Default: 20 |

### GET `/requests` Response Format

```json
{
  "items": [ ... ],
  "total": 42,
  "page": 1,
  "per_page": 20,
  "pages": 3
}
```

### PATCH `/requests/{id}` — Role-Dependent Fields

- **User (own request, not done):** `title`, `description`, `priority`, `status`
- **Manager:** `status` (not revert done), `assignee_id` (must be a manager)
- **Admin:** all fields

### Error Responses

All errors follow a consistent format:
```json
{
  "detail": "Human-readable error message"
}
```

| Code | When |
|------|------|
| 400 | Validation errors, invalid status transitions |
| 401 | Missing/invalid/expired token |
| 403 | Insufficient permissions for the action |
| 404 | Resource not found |
| 409 | Duplicate username on registration |

## 7. Backend Architecture

### Project Structure

```
backend/
├── Dockerfile
├── pyproject.toml
├── alembic.ini
├── alembic/
│   └── versions/
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI app factory, lifespan (seed), CORS
│   ├── config.py            # Pydantic BaseSettings (env vars)
│   ├── database.py          # async engine, async_sessionmaker, Base
│   ├── models/
│   │   ├── __init__.py
│   │   ├── user.py          # User SQLAlchemy model
│   │   └── request.py       # Request SQLAlchemy model
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── user.py          # UserCreate, UserResponse, RoleUpdate
│   │   ├── request.py       # RequestCreate, RequestUpdate, RequestResponse, RequestListResponse
│   │   └── auth.py          # TokenResponse, LoginRequest
│   ├── api/
│   │   ├── __init__.py      # api_router with all sub-routers
│   │   ├── auth.py          # Auth endpoints
│   │   ├── users.py         # User management endpoints
│   │   └── requests.py      # Request CRUD endpoints
│   ├── services/
│   │   ├── __init__.py
│   │   ├── auth.py          # hash_password, verify_password, create_tokens, decode_token
│   │   ├── user.py          # get_user, list_users, update_role
│   │   └── request.py       # create, get, list (with filters), update, delete + permission validation
│   ├── dependencies/
│   │   ├── __init__.py
│   │   ├── auth.py          # get_current_user, require_role(role) → Depends
│   │   └── database.py      # get_db → AsyncSession
│   └── seed.py              # Create admin:admin if not exists
└── tests/
    ├── conftest.py          # async test fixtures: in-memory SQLite, test client, auth helpers
    ├── test_auth.py         # Registration, login, refresh, logout
    ├── test_requests.py     # CRUD operations, filters, pagination
    └── test_permissions.py  # Role-based access control matrix
```

### Key Design Decisions

- **Async throughout:** `async def` endpoints, `AsyncSession`, `aiosqlite` — modern FastAPI best practice.
- **Thin routers:** Routers only handle HTTP concerns (parse request, call service, return response). All business logic lives in `services/`.
- **Dependency injection:** `get_current_user` and `require_role()` as FastAPI dependencies. Composable and testable.
- **Service layer returns domain objects**, routers convert to Pydantic response schemas.
- **CORS:** Configured in `main.py` for frontend origin (configurable via env).

## 8. Frontend Architecture

### Project Structure

```
frontend/
├── Dockerfile
├── nginx.conf
├── package.json
├── vite.config.ts
├── tsconfig.json
├── tailwind.config.ts
├── components.json              # shadcn/ui
├── src/
│   ├── main.tsx                 # React DOM entry
│   ├── App.tsx                  # QueryClientProvider + RouterProvider
│   ├── router.tsx               # Route definitions
│   ├── shared/
│   │   ├── api/
│   │   │   ├── client.ts        # Axios instance, interceptors, refresh logic
│   │   │   └── types.ts         # PaginatedResponse<T>, ApiError
│   │   ├── ui/                  # shadcn/ui components
│   │   ├── lib/
│   │   │   └── utils.ts         # cn() helper
│   │   └── hooks/
│   │       └── use-debounce.ts  # Debounced search input
│   ├── features/
│   │   ├── auth/
│   │   │   ├── api/
│   │   │   │   ├── auth.api.ts           # login, register, refresh, logout
│   │   │   │   └── auth.queries.ts       # useMutation hooks
│   │   │   ├── store/
│   │   │   │   └── auth.store.ts         # Zustand: user, accessToken, setAuth, logout
│   │   │   ├── ui/
│   │   │   │   ├── LoginForm.tsx
│   │   │   │   └── RegisterForm.tsx
│   │   │   └── guards/
│   │   │       ├── ProtectedRoute.tsx    # Redirect to /login if not authed
│   │   │       └── GuestRoute.tsx        # Redirect to / if authed
│   │   ├── requests/
│   │   │   ├── api/
│   │   │   │   ├── requests.api.ts       # API calls
│   │   │   │   └── requests.queries.ts   # useQuery, useMutation hooks
│   │   │   ├── ui/
│   │   │   │   ├── RequestList.tsx        # Card grid / list with pagination
│   │   │   │   ├── RequestCard.tsx        # Single request card
│   │   │   │   ├── RequestFilters.tsx     # Search + dropdowns + sort
│   │   │   │   ├── RequestForm.tsx        # Create/Edit dialog (role-aware)
│   │   │   │   └── RequestStatusBadge.tsx # Colored status badge
│   │   │   └── types.ts
│   │   └── admin/
│   │       ├── api/
│   │       │   ├── users.api.ts
│   │       │   └── users.queries.ts
│   │       └── ui/
│   │           ├── UserList.tsx
│   │           └── RoleSelect.tsx
│   ├── pages/
│   │   ├── LoginPage.tsx
│   │   ├── RegisterPage.tsx
│   │   ├── DashboardPage.tsx
│   │   └── AdminPage.tsx
│   └── styles/
│       └── globals.css
└── tests/
    ├── setup.ts
    ├── auth.test.tsx
    └── requests.test.tsx
```

### Routing

| Path | Component | Guard | Access |
|------|-----------|-------|--------|
| `/login` | LoginPage | GuestRoute | Unauthenticated only |
| `/register` | RegisterPage | GuestRoute | Unauthenticated only |
| `/` | DashboardPage | ProtectedRoute | All authenticated |
| `/admin` | AdminPage | ProtectedRoute + admin check | Admin only |

### State Management Strategy

- **Zustand** (auth only): `{ user: User | null, accessToken: string | null, setAuth, logout }`. No persistence — access token lives only in memory, refresh token in httpOnly cookie.
- **TanStack Query** (server state): All API data. Query keys scoped by feature: `['requests', filters]`, `['users']`. Automatic cache invalidation on mutations.

### Axios Interceptor Architecture

```
Request Interceptor:
  → Attach Authorization header from Zustand

Response Interceptor (401 handling):
  → If refreshing → queue request in pending queue
  → If not refreshing → set isRefreshing flag → call /auth/refresh
  → On success → update Zustand → retry all queued requests
  → On failure → logout → redirect to /login
```

### UI/UX Details

- **Loading states:** Skeleton loaders (shadcn Skeleton component) for lists and cards.
- **Empty states:** Friendly message + illustration when no requests found.
- **Error handling:** Toast notifications (shadcn Sonner/Toast) for all API errors.
- **Responsive:** Mobile-friendly layout with collapsible filters.
- **Debounced search:** 300ms debounce on search input to avoid excessive API calls.

## 9. Docker & Infrastructure

### Backend Dockerfile (multi-stage)

```dockerfile
# Stage 1: Builder
FROM python:3.12-slim AS builder
WORKDIR /app
COPY pyproject.toml ./
RUN pip install --no-cache-dir .

# Stage 2: Runtime
FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin
COPY . .
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Frontend Dockerfile (multi-stage)

```dockerfile
# Stage 1: Builder
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Runtime
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

### nginx.conf

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;    # SPA fallback
    }

    location /api/ {
        proxy_pass http://backend:8000/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### docker-compose.yml (Production)

```yaml
version: "3.8"
services:
  backend:
    build: ./backend
    ports: ["8000:8000"]
    volumes: ["./data:/app/data"]
    environment:
      - DATABASE_URL=sqlite+aiosqlite:///./data/app.db
      - JWT_SECRET=${JWT_SECRET}
      - CORS_ORIGINS=http://localhost:3000

  frontend:
    build: ./frontend
    ports: ["3000:80"]
    depends_on: [backend]
```

### docker-compose.dev.yml (Development)

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
    environment:
      - VITE_API_URL=http://localhost:8000/api
    command: npm run dev -- --host
    depends_on: [backend]
```

## 10. CI/CD (GitHub Actions)

### Backend CI (`.github/workflows/ci.yml` in backend repo)

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
      - run: pip install -e ".[dev]"
      - run: ruff check .
      - run: ruff format --check .
      - run: pytest -v
```

### Frontend CI (`.github/workflows/ci.yml` in frontend repo)

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
      - run: npm ci
      - run: npm run lint
      - run: npx tsc --noEmit
      - run: npm run test -- --run
```

## 11. Testing Strategy

### Backend Tests (Pytest)

- **Fixtures:** In-memory SQLite (`sqlite+aiosqlite:///:memory:`), async test client via `httpx.AsyncClient`, helper functions for creating authenticated users of each role.
- **Test categories:**
  - `test_auth.py` — register (success, duplicate), login (success, wrong creds), refresh (success, expired), logout
  - `test_requests.py` — CRUD operations, search/filter/sort/pagination edge cases
  - `test_permissions.py` — Full permission matrix: each role × each action, including edge cases (edit done request, revert done, assign to non-manager)

### Frontend Tests (Vitest)

- **Setup:** `@testing-library/react`, MSW for API mocking
- **Test categories:**
  - `auth.test.tsx` — Form validation, login/register flows, error display
  - `requests.test.tsx` — List rendering, filter interactions, form submission

## 12. Configuration & Environment Variables

### Backend (`app/config.py` via Pydantic BaseSettings)

| Variable | Default | Description |
|----------|---------|-------------|
| `DATABASE_URL` | `sqlite+aiosqlite:///./data/app.db` | Database connection string |
| `JWT_SECRET` | (required) | Secret key for JWT signing |
| `JWT_ACCESS_EXPIRE_MINUTES` | `15` | Access token lifetime |
| `JWT_REFRESH_EXPIRE_DAYS` | `7` | Refresh token lifetime |
| `CORS_ORIGINS` | `http://localhost:5173` | Allowed CORS origins (comma-separated) |
| `ADMIN_USERNAME` | `admin` | Default admin username |
| `ADMIN_PASSWORD` | `admin` | Default admin password |

### Frontend (Vite env vars)

| Variable | Default | Description |
|----------|---------|-------------|
| `VITE_API_URL` | `/api` | Backend API base URL |
