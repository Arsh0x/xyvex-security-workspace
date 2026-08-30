# XYVEX Release Engineering & Testing Specification

This document details the release engineering pipeline, container orchestration, database migration strategy, and automated testing suites built for **XYVEX**.

---

## 1. Zero-Downtime Safe Database Migrations (`migration_bootstrap.py`)

A major engineering milestone in XYVEX was designing a database initialization mechanism capable of handling both fresh database deployments and existing populated legacy database instances without data corruption.

```text
                           init_db.py Executed
                                    │
                                    ▼
                       Inspect Database State
                                    │
            ┌───────────────────────┴───────────────────────┐
            ▼                                               ▼
   Fresh Database Detected                       Legacy Database Detected
            │                                               │
   Run Alembic Upgrade Head                         Inspect Existing Tables
            │                                               │
            │                                     Acquire Postgres Advisory Lock
            │                                               │
            │                                     Mark Legacy Baseline Revision
            │                                               │
            └───────────────────────┬───────────────────────┘
                                    ▼
                         Database Ready & Safe
```

### Key Technical Aspects of `migration_bootstrap.py`

1. **Dialect & Schema Reflection**: Inspects PostgreSQL / SQLite metadata to detect pre-existing tables (`users`, `challenges`, `commands`).
2. **PostgreSQL Advisory Locking**: Uses `pg_advisory_xact_lock(0x5859564558)` to ensure concurrent application worker containers during a rolling deploy do not execute duplicate bootstrap migrations simultaneously.
3. **Atomic Migration Adoption**: If legacy tables are detected, `migration_bootstrap.py` inserts the baseline Alembic revision record into `alembic_version` before triggering `alembic upgrade head`, preventing destructive duplicate table creation errors.

---

## 2. Production Docker Architecture & Deployment Stack

XYVEX uses multi-stage Docker builds to ensure minimal image sizes, zero development artifacts in production, and deterministic build reproducibility.

```text
docker-compose.prod.yml
├── service: postgres (PostgreSQL 16 Alpine, Healthcheck via pg_isready)
├── service: backend (Python 3.12 Slim, executes start.sh -> Gunicorn/Uvicorn)
└── service: nginx (Nginx 1.25 Alpine, static dist assets + TLS reverse proxy)
```

### `Dockerfile` Multi-Stage Strategy

```dockerfile
# Stage 1: Build Frontend Assets
FROM node:22-alpine AS frontend-builder
WORKDIR /app/frontend
COPY frontend/package*.json ./
RUN npm ci
COPY frontend/ ./
RUN npm run build

# Stage 2: Production Runtime Backend & Gateway
FROM python:3.12-slim AS backend-runtime
WORKDIR /app
COPY requirements.lock ./
RUN pip install --no-cache-dir -r requirements.lock
COPY . .
COPY --from=frontend-builder /app/frontend/dist /app/static

EXPOSE 8000
CMD ["/bin/bash", "start.sh"]
```

### `start.sh` Container Entrypoint Execution Flow

```bash
#!/usr/bin/env bash
set -eo pipefail

echo "[*] Awaiting PostgreSQL database connectivity..."
python3 -c "import socket, time; ... socket.create_connection(('postgres', 5432))"

echo "[*] Running database migration bootstrap..."
python3 migration_bootstrap.py

echo "[*] Starting production Uvicorn application server..."
exec uvicorn app:app --host 0.0.0.0 --port 8000 --workers 4 --proxy-headers
```

---

## 3. Automated Quality Assurance & Verification Suite

XYVEX maintains a multi-layered verification matrix to guarantee application stability and security isolation across updates.

```text
┌─────────────────────────────────────────────────────────────────┐
│                      Automated E2E Suite                        │
│             Playwright (Chromium / Firefox / WebKit)            │
│         Verifies Login, Workspace, Terminal & Writeups          │
└────────────────────────────────┬────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Backend Regression Suite                      │
│                  Pytest + Async HTTPX Client                    │
│      Auth, Cryptography, Multi-tenant Isolation, Parameters     │
└────────────────────────────────┬────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Database Migration Tests                     │
│                  Alembic Rollback & Adoption                    │
│          Clean Upgrade, Downgrade & Data Validation             │
└─────────────────────────────────────────────────────────────────┘
```

### Test Coverage Highlights

1. **Backend Regression Suite (`pytest`)**:
   - Covers user registration, email verification token generation, password reset flows, JWT expiration, multi-tenant 404 security checks, Fernet encryption integrity, and command parameterization logic.
   - Run via: `pytest tests/ -v --cov=.`

2. **End-to-End Browser Suite (`playwright`)**:
   - Executes headful and headless browser workflows simulating real CTF operator sequences.
   - Tests target creation, service port addition, terminal command execution log tagging, parameter promotion to the library, and markdown writeup export.
   - Run via: `npx playwright test`

3. **Disaster Recovery & Backup Verification**:
   - Automated verification script (`scripts/verify_backup_restore.sh`) dumps the active PostgreSQL database, resets the container environment, restores the SQL dump into a clean instance, and executes state verification checks to ensure zero data loss.
