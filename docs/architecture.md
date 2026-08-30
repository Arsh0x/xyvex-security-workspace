# XYVEX Architecture Documentation

This document provides a comprehensive technical overview of the **XYVEX** platform architecture, data flow, component interactions, database schema, and deployment topology.

---

## 1. System Topology & Tiered Layout

XYVEX is architected as a high-performance 3-tier web application decoupled into a single-page React application (SPA), an asynchronous FastAPI app server, and a relational PostgreSQL database.

```text
                               ┌─────────────────────────┐
                               │  Client Web Browser     │
                               │  React 18 + TS + Vite   │
                               └────────────┬────────────┘
                                            │ HTTPS / WSS
                                            ▼
                               ┌─────────────────────────┐
                               │     Nginx Gateway       │
                               │ Static Server & Proxy   │
                               └────────────┬────────────┘
                                            │ Reverse Proxy / Unix Socket
                                            ▼
                               ┌─────────────────────────┐
                               │   FastAPI App Server    │
                               │ Python 3.12 + AsyncIO   │
                               └────────────┬────────────┘
                                            │ SQLAlchemy 2.0 Async
                                            ▼
                               ┌─────────────────────────┐
                               │  PostgreSQL Database    │
                               │  Relational Storage     │
                               └─────────────────────────┘
```

### Component Roles & Boundaries

1. **Client Tier (Frontend)**:
   - **Framework**: React 18, TypeScript 5, Vite, Tailwind CSS.
   - **State & Router**: React Router v6, custom hook context providers for auth state, active challenge sessions, and execution logs.
   - **Responsibility**: Instant client-side render, keyboard-shortcut-driven terminal interface, real-time command syntax highlighting, evidence previewing, and interactive markdown writeup authoring.

2. **API Gateway & Static Proxy (Nginx)**:
   - **TLS Termination**: Offloads HTTPS encryption using modern cipher suites (`TLSv1.2`, `TLSv1.3`).
   - **Static Content**: Direct high-efficiency static file delivery for compiled Vite production assets (`/dist`).
   - **Request Forwarding**: Proxies API traffic to backend Gunicorn/Uvicorn workers (`/api/v1/*`) with custom header propagation (`X-Forwarded-For`, `X-Forwarded-Proto`).
   - **Security Headers**: Enforces strict CSP, HSTS, X-Content-Type-Options, and X-Frame-Options policies.

3. **Application Server Tier (Backend)**:
   - **Framework**: FastAPI (Python 3.12) running under Uvicorn worker process pool.
   - **Domain Services**: Modular service layer segregating business logic (`AuthService`, `ChallengeService`, `KnowledgeLibraryService`, `RedactionService`, `ExportService`).
   - **Data Validation**: Strict Pydantic v2 schemas validating incoming request payloads and enforcing precise JSON output shapes.

4. **Data Persistence Tier (Database)**:
   - **Engine**: PostgreSQL 16.
   - **ORM**: Async SQLAlchemy 2.0 with connection pooling (`asyncpg`).
   - **Migrations**: Alembic version control with atomic transaction locks.

---

## 2. Database Schema & Entity Relationships

The XYVEX database model is engineered around a strict hierarchical multi-tenant structure anchored by the `User` and `Challenge` entities.

```text
┌──────────────┐       1:N       ┌──────────────────┐
│    Users     ├─────────────────►    Challenges    │
└──────┬───────┘                 └────────┬─────────┘
       │                                  │ 1:N
       │ 1:N                              ├──────────────────────┬──────────────────────┐
       ▼                                  ▼                      ▼                      ▼
┌──────────────┐                 ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│ RefreshTokens│                 │      Hosts       │   │    Checklists    │   │     Writeups     │
└──────────────┘                 └────────┬─────────┘   └──────────────────┘   └──────────────────┘
                                          │ 1:N
                                          ├──────────────────────┐
                                          ▼                      ▼
                                 ┌──────────────────┐   ┌──────────────────┐
                                 │     Services     │   │   Credentials    │
                                 └────────┬─────────┘   └──────────────────┘
                                          │ 1:N
                                          ▼
                                 ┌──────────────────┐
                                 │     Commands     │
                                 └────────┬─────────┘
                                          │ 1:N
                                          ▼
                                 ┌──────────────────┐
                                 │ ExecutionAttempts│
                                 └────────┬─────────┘
                                          │ 1:N
                                          ▼
                                 ┌──────────────────┐
                                 │     Evidence     │
                                 └──────────────────┘
```

### Core Schema Entity Specifications

#### `users`
* `id` (UUID, Primary Key)
* `email` (String, Unique, Indexed)
* `password_hash` (String, Argon2id / PBKDF2-SHA256)
* `is_verified` (Boolean, Default: False)
* `created_at` (Timestamp UTC)

#### `challenges`
* `id` (UUID, Primary Key)
* `user_id` (UUID, Foreign Key -> `users.id`, Indexed)
* `title` (String)
* `platform` (String, e.g., 'Hack The Box', 'TryHackMe', 'OffSec')
* `target_ip` (String)
* `difficulty` (Enum: Easy, Medium, Hard, Insane)
* `status` (Enum: In Progress, Rooted, Archived)
* `created_at` (Timestamp UTC)

#### `hosts`
* `id` (UUID, Primary Key)
* `challenge_id` (UUID, Foreign Key -> `challenges.id`, Indexed)
* `hostname` (String)
* `ip_address` (String)
* `os_family` (String, e.g., 'Linux', 'Windows')

#### `services`
* `id` (UUID, Primary Key)
* `host_id` (UUID, Foreign Key -> `hosts.id`, Indexed)
* `port` (Integer)
* `protocol` (String, e.g., 'tcp', 'udp')
* `service_name` (String, e.g., 'smb', 'http', 'ssh')
* `version_banner` (Text)

#### `commands` & `execution_attempts`
* `id` (UUID, Primary Key)
* `service_id` (UUID, Foreign Key -> `services.id`, Indexed)
* `raw_command` (Text)
* `output_snippet` (Text)
* `status_code` (Integer)
* `promoted_to_library` (Boolean)

#### `credentials` & `evidence`
* `id` (UUID, Primary Key)
* `challenge_id` (UUID, Foreign Key -> `challenges.id`)
* `secret_data_encrypted` (Text, Fernet AES-256 payload)
* `description` (String)

---

## 3. Asynchronous Execution & IO Strategy

XYVEX handles high-frequency logging of CLI commands, output stream chunks, and concurrent API requests.

1. **Non-Blocking Database IO**:
   - SQLAlchemy 2.0 async engine (`async sessionmaker`) ensures HTTP worker threads are never blocked while awaiting PostgreSQL IO.
   - Connection pools are configured with `pool_size=20`, `max_overflow=10`, and `pool_pre_ping=True` to recover gracefully from connection idle timeouts.

2. **Bulk Command Ingestion**:
   - Log streams are ingested via structured batch endpoints (`POST /api/v1/challenges/{id}/commands/batch`).
   - Transaction boundaries are strictly scoped to ensure partial execution log batches can be rolled back safely without corrupting host state.

---

## 4. Frontend State Management & API Client

- **Optimistic UI Updates**: Command attempt logs and tag modifications trigger immediate visual UI state changes before network response resolution, reverting smoothly if a backend validation error occurs.
- **Axios HTTP Interceptor Pipeline**:
  - Automatically injects `Authorization: Bearer <access_token>` headers.
  - Intercepts `401 Unauthorized` responses to execute silent background JWT token refresh via `/api/v1/auth/refresh`.
  - Queues failed requests while refresh is pending, re-issuing them seamlessly upon token renewal.
