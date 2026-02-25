# BACKEND.md — Python/FastAPI Backend
## ScandiTech Digital Installation Standard (DIS) — Fire Protection Edition

This document defines the authoritative backend architecture and engineering rules
for DIS using **Python + FastAPI**. It is designed for deterministic rule evaluation,
versioned solution publishing, and long-term scalability.

---

## 1) Purpose

DIS backend provides:
- Deterministic evaluation of fire protection scenarios
- Multi-manufacturer compliant solution listing (**participating manufacturers only**)
- Standardized outputs: SSR + requirements + steps + materials + references
- ScandiTech-only publishing workflow (no manufacturer login/dashboard)
- Exportable Solution Pack (PDF)

DIS backend does **not** certify installation execution (solution reference only).

---

## 2) Tech Stack (Locked)

### Runtime
- Python 3.12

### API Framework
- FastAPI
- Uvicorn (dev), Gunicorn + UvicornWorker (prod)

### Database
- PostgreSQL

### ORM + Migrations (recommended)
- SQLAlchemy 2.0 (async)
- Alembic

> If async SQLAlchemy is too heavy for your team, use synchronous SQLAlchemy with a
> threadpool. But keep repository boundaries identical.

### Queue / Background Jobs
- Redis
- Dramatiq (preferred) **or** Celery (acceptable if already used)

### File Storage
- S3-compatible (AWS S3 / MinIO)
- boto3

### PDF Generation
- Playwright (Python) HTML → PDF

### Validation / Schemas
- Pydantic v2 (FastAPI native)

### Docs
- OpenAPI built into FastAPI

### Logging & Monitoring
- structlog (recommended) or std logging with JSON formatter
- Sentry (optional but recommended)

---

## 3) Core Architectural Principles (Hard Rules)

1. **Rule Engine is deterministic**. No AI selection in core evaluation.
2. **No eval**. Rule computations use a safe DSL interpreter with whitelisted operators.
3. **Published SolutionVersions are immutable**. Edits create a new version.
4. **ScandiTech is the publisher**. Manufacturers are data entities only.
5. **Compliance logic is server-side only**. Clients are presentation layers.
6. **Everything is versioned and auditable**. Admin content changes create audit logs.

---

## 4) Repo Layout (Required)

Use this structure to prevent spaghetti growth:
app/
main.py
core/
config.py
logging.py
security.py
deps.py
errors.py
api/
v1/
routers/
evaluate.py
solutions.py
exports.py
admin_manufacturers.py
admin_products.py
admin_solutions.py
init.py
schemas/
common.py
scenario.py
solution.py
admin.py
domain/
rule_engine/
evaluator.py
operators.py
lookup.py
case.py
constraints.py
ssr.py
matching.py
types.py
publishing/
versioning.py
workflow.py
services/
storage_s3.py
pdf_export.py
queue.py
repositories/
manufacturers.py
products.py
solutions.py
solution_versions.py
audit.py
db/
session.py
models/
base.py
user.py
organization.py
manufacturer.py
product.py
solution.py
solution_version.py
audit_log.py
migrations/ (alembic)
tests/
unit/
integration/


**Rule:** Routers should contain no business logic. Domain services do not import FastAPI.

---

## 5) Domain Model (Publisher-Controlled)

### Core entities
- Manufacturer (data entity only)
- Product (catalog, optional but recommended)
- Solution (draft container)
- SolutionVersion (immutable published snapshot)
- Applicability (filters)
- Rules (DSL JSON)
- Steps
- Materials (BOM + quantity logic)
- References (documents)
- AuditLog

### Publishing state
- DRAFT → IN_REVIEW → PUBLISHED → ARCHIVED

Publishing creates a new SolutionVersion. Only one active version per Solution.

---

## 6) Rule Engine (The Brain)

### Responsibilities
Given a scenario payload:
1. Validate payload via Pydantic schema per `scenario_type`
2. Normalize to typed `vars` map (units included)
3. Fetch candidate SolutionVersions for scenario_type + active versions
4. Apply applicability filtering (construction, fire rating, service type, shape, size ranges)
5. Compute derived outputs via DSL:
   - `hole_diameter_mm`
   - tolerances
   - clearances
   - insulation requirements
6. Evaluate constraints (warnings/errors)
7. Generate SSR preview for each solution
8. Return solution cards

### Output is explainable
Engine should be able to return (optional) debug info:
- why solution excluded (admin-only)
- which applicability rule failed

---

## 7) DSL Rule JSON (Safe Expression Interpreter)

Rules are stored as JSON and evaluated by a safe interpreter.

### Allowed operator categories (whitelist)
- arithmetic: add, sub, mul, div, min, max, round, ceil, floor
- comparison: lt, lte, gt, gte, eq, in
- boolean: and, or, not
- value: const, var
- control: case
- lookup: lookup

**No freeform code. No function calls beyond whitelist.**

Constraints are:
- `when` (boolean expr)
- `assert` (boolean expr)
- severity: WARN/ERROR
- message: string

---

## 8) SSR (Standard Solution Reference)

### Goal
Stable solution-level reference code for documentation.

### Rules
- Generated from `SolutionVersion.ssr_template`
- Substitutes placeholders from scenario + solution metadata
- Represents solution configuration only (not installation event)

---

## 9) API Surface (Minimum)

### Public (Customer)
- `POST /v1/evaluate`
- `GET /v1/solutions/{solution_version_id}`
- `POST /v1/exports/solution-pack`

### Admin (ScandiTech only)
- Manufacturers CRUD
- Products CRUD
- Solutions draft CRUD
- Publish endpoint: `POST /v1/admin/solutions/{solution_id}/publish`

OpenAPI must be accurate and kept up to date.

---

## 10) Authentication & RBAC

Use JWT access + refresh tokens.

### Roles
Internal:
- SUPER_ADMIN
- CONTENT_ADMIN
- REVIEWER

External:
- CUSTOMER_ADMIN
- FIELD_USER

Admin routes require internal roles only.

---

## 11) Export Pipeline (PDF Solution Pack)

### Approach
- Build HTML template (Jinja2 or equivalent)
- Render in Playwright → PDF
- Upload PDF to S3
- Return signed URL

### Queue
Use Dramatiq/Celery for:
- PDF generation
- bulk imports
- indexing jobs (future)

Ensure idempotency:
- `export_key = hash(solutionVersionId + scenario_payload + template_version)`

---

## 12) Data Access Pattern (Repositories)

All DB access must go through `repositories/*`.
Domain services call repositories; routers call domain services.

This enforces testability and prevents API-layer coupling.

---

## 13) Testing Strategy

### Unit Tests
- DSL evaluator (expressions)
- case and lookup
- constraints evaluation
- SSR generator
- applicability matcher

### Integration Tests
- Evaluate endpoint returns correct results with seeded data
- Solutions detail endpoint returns correct structure
- Export job enqueues and completes (mock S3/playwright where needed)

Use:
- pytest
- httpx TestClient
- Testcontainers (optional but recommended for CI)

---

## 14) Environment Variables (Minimum)

- DATABASE_URL
- REDIS_URL
- S3_BUCKET
- S3_REGION
- S3_ENDPOINT (if MinIO)
- S3_ACCESS_KEY_ID
- S3_SECRET_ACCESS_KEY
- JWT_ACCESS_SECRET
- JWT_REFRESH_SECRET
- PUBLIC_BASE_URL
- SENTRY_DSN (optional)

---

## 15) Immediate Deliverables (Backend Team)

1. FastAPI skeleton with routing + DI patterns
2. SQLAlchemy models + Alembic migrations
3. Seed script:
   - ScandiTech admin user
   - 1–2 manufacturers
   - products
   - at least 1 published SolutionVersion with full rules
4. Rule Engine module:
   - applicability match
   - DSL compute outputs
   - constraints evaluation
   - SSR generation
5. Endpoints:
   - /evaluate
   - /solutions/{solutionVersionId}
   - /exports/solution-pack

---

## 16) Implementation “Do Not Do”

- Do not implement manufacturer logins.
- Do not implement AI selection.
- Do not mutate published solution versions.
- Do not store formulas as executable strings.
- Do not duplicate rule logic in frontend.

---

## 17) Next Steps After Base Build

- Define Pydantic scenario schemas for each scenario_type
- Expand admin authoring UI requirements
- Build import tools for internal digitization workflows
- Add full audit coverage and review workflow enforcement