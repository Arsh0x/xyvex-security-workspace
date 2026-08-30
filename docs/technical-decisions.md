# XYVEX Architectural Decision Records (ADRs)

This document records the key architectural decisions, design trade-offs, and technology evaluations made during the engineering of **XYVEX**.

---

## ADR 001: Backend Framework — FastAPI vs Django / Flask

### Context & Problem Statement
The XYVEX application server requires high-throughput asynchronous execution logging, strict input schema validation, automatic OpenAPI spec generation, and low response latency for interactive terminal sessions.

### Decision
We selected **FastAPI** (running on Python 3.12 with Uvicorn and Pydantic v2).

### Evaluation & Rationale
* **FastAPI vs Flask**: Flask lacks native async support and built-in type enforcement, requiring multiple third-party libraries (Marshmallow, Webargs, Flask-RESTful) that increase maintenance friction. FastAPI delivers native `async/await` handling out of the box.
* **FastAPI vs Django**: Django provides a battery-included framework, but its ORM is historically synchronous and overly heavy for a REST API micro-architecture. FastAPI paired with SQLAlchemy 2.0 Async gives granular control over query execution and payload schemas.
* **Performance**: FastAPI's Starlette core and Pydantic C-compiled core (`pydantic-core`) deliver request parsing speeds comparable to Node.js / Go for JSON-heavy workloads.

---

## ADR 002: Frontend Architecture — React 18 + Vite SPA vs Next.js SSR

### Context & Problem Statement
XYVEX is a desktop-grade security workspace requiring persistent client-side terminal state, rapid state mutations, offline-first command drafting, and instant tab transitions.

### Decision
We selected a **Single-Page Application (SPA)** architecture using **React 18, TypeScript 5, and Vite**, served statically via Nginx.

### Evaluation & Rationale
* **SPA vs Next.js SSR**: Server-Side Rendering (SSR) in Next.js is optimized for public content SEO and e-commerce. XYVEX is an authenticated application workspace behind login controls. SSR adds server runtime complexity, hydration edge-cases with terminal execution logs, and unnecessary node.js server dependencies in production.
* **Vite Build Performance**: Vite provides instantaneous Hot Module Replacement (HMR) during development and highly optimized Rollup bundles for production static serving via Nginx.

---

## ADR 003: Data Layer — PostgreSQL + Async SQLAlchemy 2.0 vs NoSQL (MongoDB)

### Context & Problem Statement
Security challenge data consists of interconnected hierarchical entities: Challenges, Hosts, Ports, Services, Commands, Attempts, Credentials, and Evidence.

### Decision
We selected **PostgreSQL 16** with **Async SQLAlchemy 2.0** and **Alembic**.

### Evaluation & Rationale
* **Relational vs Document Store**: A NoSQL document store (MongoDB) might initially seem flexible for unstructured command outputs. However, relational integrity, foreign key cascading deletes (e.g., deleting a Host automatically purges child Services and Commands), and transactional consistency are critical to avoid orphan records.
* **JSONB Capabilities**: PostgreSQL provides native `JSONB` columns for storing arbitrary tool outputs or Nmap XML scan parses, giving us the flexibility of a document store alongside relational ACID guarantees.

---

## 4. Architectural Trade-offs Summary Table

| Category | Option Chosen | Alternative Considered | Trade-off Rationale |
| :--- | :--- | :--- | :--- |
| **Authentication** | Dual JWT + Refresh Cookie | Session-based Cookies | Enables stateless API scalability across microservices while retaining instant token revocation capability. |
| **Field Encryption** | Fernet AES-256 (App Layer) | DB-level `pgcrypto` / KMS | Application-layer encryption ensures database administrators or compromised SQL dumps cannot read credentials in plaintext. |
| **Styling Engine** | Vanilla CSS + Tailwind CSS | CSS-in-JS (Styled-Components) | Eliminates runtime CSS parsing overhead, providing zero-runtime bundle efficiency for dense UI widgets. |
| **Testing Strategy** | Pytest + Playwright | Selenium / Cypress | Playwright provides faster, headless multi-browser execution with native network interception capabilities. |
