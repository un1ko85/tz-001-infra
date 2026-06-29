# Issue Tracker — Infrastructure

Docker Compose configuration and shared infrastructure for the Issue Tracker application.

## Repositories

| Repository | Description |
|------------|-------------|
| `backend/` | FastAPI + SQLite + SQLAlchemy |
| `frontend/` | React + TypeScript + Vite + shadcn/ui |

## Quick Start

```bash
# Development
docker compose -f docker-compose.dev.yml up --build

# Production
docker compose up --build
```

## Structure

```
├── docker-compose.yml       # Production config
├── docker-compose.dev.yml   # Development config with hot-reload
├── backend/                 # Backend repo (separate git)
├── frontend/                # Frontend repo (separate git)
└── data/                    # SQLite database (gitignored)
```
