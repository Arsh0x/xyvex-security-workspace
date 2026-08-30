# XYVEX — Public Engineering Case Study

> **XYVEX** is a cybersecurity workspace designed and engineered for CTFs, security labs, and hands-on learning. It structures commands, execution attempts, credentials, evidence, workflows, and writeups while solving a challenge, then turns useful discoveries into reusable knowledge for future challenges.
>
> **Production source code is private because XYVEX is being prepared as a commercial product. This repository is a public engineering case study documenting the architecture, system design decisions, security model, release engineering, testing strategy, and technical metrics.**

---

## 🔄 Core Product Loop

XYVEX transforms chaotic CTF notes and command-line interactions into structured, repeatable knowledge across security challenges:

```text
Challenge A (e.g. Cicada / HTB)
    │
    ├─► Capture Commands, Service Scans & Proof Evidence
    │
    ├─► Parameterize Useful Commands (e.g. smbclient -L //<TARGET_IP> -N)
    │
    └─► Promote to Knowledge Library
            │
            ▼
     Global Knowledge Library
            │
            ▼
Challenge B (e.g. Resolute / HTB)
    │
    └─► Contextual Recommendation Engine surfaces past SMB command templates
```

---

## 🏗️ Technical Architecture Overview

XYVEX is built as a highly responsive, multi-tenant web application engineered for speed, strict security boundaries, and reliable offline/online workflow tracking.

```text
┌─────────────────────────────────────────────────────────────────┐
│                      Client Layer (Browser)                     │
│           React 18 + TypeScript + Vite + Tailwind CSS           │
└────────────────────────────────┬────────────────────────────────┘
                                 │ REST API / JSON over TLS
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Reverse Proxy / Gateway                     │
│                        Nginx (TLS / SSL)                        │
└────────────────────────────────┬────────────────────────────────┘
                                 │ Forward Request
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Application Server Layer                    │
│                 FastAPI (Python 3.12) + Pydantic                │
│                                                                 │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────┐ │
│  │ Challenge Authz   │  │ Fernet Encryption│  │ JWT Rotation  │ │
│  │ Middleware       │  │ Engine           │  │ Manager       │ │
│  └──────────────────┘  └──────────────────┘  └───────────────┘ │
└────────────────────────────────┬────────────────────────────────┘
                                 │ Async SQLAlchemy ORM
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Database Layer                          │
│                   PostgreSQL 16 + Alembic                       │
└─────────────────────────────────────────────────────────────────┘
```

For full architectural details, component breakdown, and database schemas, see [docs/architecture.md](docs/architecture.md).

---

## 📊 Verification Matrix & Test Evidence

Every release candidate undergoes multi-stage verification including automated backend regression testing, database migration validation, and Playwright end-to-end browser testing.

| Area | Verification Method | Status | Target Benchmark |
| :--- | :--- | :---: | :--- |
| **Authentication & JWT** | Pytest unit & rotation tests | ✅ | Password hashing, token refresh & blacklisting verified |
| **Email Verification** | Integration test + Resend API | ✅ | Verified link generation & state transition |
| **Password Reset** | Expiration & single-use test | ✅ | Token single-use enforcement verified |
| **Cross-User Isolation** | Ownership middleware tests | ✅ | Zero cross-tenant data leakage (404 enforcement) |
| **Sensitive Data Encryption** | Fernet AES-256 field test | ✅ | Credentials & proofs encrypted at rest |
| **Library Reuse Engine** | Parameter substitution engine | ✅ | Automated variable template generation |
| **Writeup Redaction** | Regex & token secret scrubbing | ✅ | Credential masking prior to export verified |
| **Browser E2E Workflow** | Playwright test suite | ✅ | Full CTF workflow execution passing |
| **Schema Migrations** | Alembic upgrade/downgrade | ✅ | Atomic migrations & legacy SQLite adoption |
| **Disaster Recovery** | Automated dump & restore | ✅ | Full data integrity post-restoration verified |

---

## 🖼️ Application Screenshots & UI Showcase

The workspace provides an intuitive, high-density interface tailored for fast command tracking and knowledge extraction:

- 📊 **[Overview / Dashboard](screenshots/overview.png)**: Active challenges, target hosts, platform metrics, and recent activity.
- 📦 **[Boxes / Target Hub](screenshots/boxes.png)**: Host management, port scan ingestion, service inventory, and credential storage.
- 💻 **[Workspace Terminal Engine](screenshots/workspace.png)**: Live command execution log, outcome tagging, and attempt timeline.
- 📚 **[Knowledge Library](screenshots/library.png)**: Reusable command snippets, category filters, and parameterization engine.
- 📝 **[Writeup Generator](screenshots/writeup.png)**: Markdown editor, evidence auto-linking, and sensitive data redaction filter.
- ⚙️ **[Security & Settings](screenshots/settings.png)**: Token session management, encryption status, and account settings.

> [!NOTE]
> Detailed guidelines on screenshot sanitization and UI features are available in [screenshots/README.md](screenshots/README.md).

---

## 🧠 Deep-Dive Engineering Challenges

### 1. Zero-Downtime Safe Database Adoption (`alembic` & `migration_bootstrap.py`)
* **Problem**: Transitioning from early SQLite prototypes with populated data to production PostgreSQL schemas without data loss or downtime.
* **Solution**: Engineered `migration_bootstrap.py`, a custom migration engine that dynamically detects database engine dialect, verifies table signatures, applies advisory locks during adoption, and seamlessly bootstraps Alembic head revisions.
* **Read Detailed Writeup**: [docs/release-engineering.md](docs/release-engineering.md)

### 2. Multi-Tenant Challenge Authorization Scoping
* **Problem**: Preventing unauthorized horizontal privilege escalation where User B attempts to access User A's challenge resource via ID guessing (`GET /api/challenges/1042`).
* **Solution**: Implemented centralized database-level scoping filters in SQLAlchemy combined with authorization guards returning uniform `404 Not Found` responses to suppress resource existence leaks.
* **Read Detailed Writeup**: [docs/security-design.md](docs/security-design.md)

### 3. Field-Level Encryption & Redaction Pipeline
* **Problem**: Storing sensitive target credentials and flag proofs securely while enabling safe markdown writeup exports.
* **Solution**: Utilized Fernet symmetric encryption (AES-128-CBC with HMAC-SHA256) for data at rest, combined with a streaming redaction filter that replaces raw passwords and flags with customizable placeholder tokens (`[REDACTED_CREDENTIAL]`) during writeup compile time.
* **Read Detailed Writeup**: [docs/security-design.md](docs/security-design.md)

### 4. Dynamic Command Parameterization & Reusability Engine
* **Problem**: Converting hardcoded, machine-specific attack commands into reusable knowledge templates across different target subnets.
* **Solution**: Constructed an automated syntax parser that identifies hardcoded target IPs, hostnames, and user credentials, automatically replacing them with template tokens (e.g., `<TARGET_IP>`, `<USERNAME>`) and suggesting contextual matches on new targets.
* **Read Detailed Writeup**: [docs/product-workflow.md](docs/product-workflow.md)

---

## 📁 Case Study Repository Structure

```text
xyvex-engineering-case-study/
│
├── README.md                     # Case study overview & engineering highlights
├── LICENSE                       # Repository legal notice & terms of use
├── NOTICE                        # Product disclosure notice
│
├── docs/                         # In-depth technical documentation
│   ├── architecture.md           # System topology, database schema, Nginx gateway
│   ├── product-workflow.md       # CTF workflow domain model & library engine
│   ├── security-design.md        # Cryptography, authz, secret redaction, JWT lifecycle
│   ├── release-engineering.md    # Alembic migrations, Docker orchestration, E2E tests
│   └── technical-decisions.md    # Architecture Decision Records (ADRs) & tradeoffs
│
├── diagrams/                     # Vector architecture diagrams (SVG)
│   ├── system-architecture.svg   # 3-Tier application topology diagram
│   ├── knowledge-loop.svg        # Reusable knowledge feedback loop diagram
│   └── challenge-domain-model.svg# Domain model hierarchy diagram
│
└── screenshots/                  # Sanitized interface visual showcase
    ├── README.md                 # Visual guide & sanitization criteria
    ├── overview.png
    ├── boxes.png
    ├── workspace.png
    ├── library.png
    ├── writeup.png
    └── settings.png
```

---

## 💼 Recommended CV Entry Format

```text
XYVEX — Founder & Lead Engineer
Stack: React 18, TypeScript, FastAPI, Python 3.12, PostgreSQL, Alembic, Docker, Playwright

• Architected a full-stack cybersecurity workspace for structuring CTF reconnaissance, command execution logs, credentials, evidence, and writeups.
• Built an automated command parameterization engine that converts target-specific commands into reusable templates with contextual recommendations.
• Implemented multi-tenant challenge-scoped authorization, AES-256 field encryption for sensitive target data, JWT token rotation, and secret redaction.
• Designed a zero-downtime database migration system supporting legacy database schema adoption and Alembic versioning.
• Formulated a production Docker stack with Nginx reverse proxy, automated E2E browser testing, and staging validation.

Engineering Case Study: https://github.com/Arsh0x/xyvex-engineering-case-study
```

---

## 📄 License & Notice

This case study is published under the terms specified in [LICENSE](LICENSE). All proprietary rights to the production XYVEX product software and backend code remain reserved. See [NOTICE](NOTICE) for complete details.
