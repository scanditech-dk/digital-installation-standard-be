# AGENTS.md — Python Backend (Codex / Agent Execution Rules)
## ScandiTech Digital Installation Standard (DIS) — Fire Protection Edition

This file contains mandatory instructions for any agent (Codex) working on the DIS
Python backend repository. Follow strictly.

---

## 1) Non-Negotiable Goal

Build the backend for **ScandiTech Digital Installation Standard (DIS)**:
- Deterministic evaluation of fire protection scenarios
- Return compliant solutions from participating manufacturers only
- ScandiTech is the sole publisher (no manufacturer login/dashboard)
- Versioned publishing with immutable `SolutionVersion`
- SSR generation + exportable Solution Pack (PDF)

DIS does NOT certify installation execution. Output is solution reference only.

---

## 2) Locked Tech Stack

- Python 3.12
- FastAPI
- PostgreSQL
- SQLAlchemy 2.0
- Alembic
- Redis
- Dramatiq (preferred) or Celery
- S3-compatible storage via boto3
- Playwright (Python)
- Pydantic v2
- pytest

No alternative frameworks without approval.

---

## 3) Architecture Boundaries (Strict)

Routers:
- Define endpoints only.
- No business logic.

Domain:
- Rule engine.
- Publishing/versioning.
- SSR generation.
- Applicability matching.
- Constraint evaluation.

Repositories:
- DB interaction only.
- No business rules.

Services:
- S3 storage.
- Queue workers.
- PDF export.
- External integrations.

Domain must never import FastAPI.

---

## 4) Deterministic Rule Engine (Core Brain)

### Requirements
- Pure function logic.
- No eval.
- No dynamic execution.
- Whitelisted operators only.

### Allowed DSL Operators
Arithmetic:
- add, sub, mul, div
- min, max, round, ceil, floor

Comparisons:
- lt, lte, gt, gte, eq, in

Boolean:
- and, or, not

Value:
- const, var

Control:
- case

Lookup:
- lookup

Unknown operator → hard failure.

---

## 5) Publishing Rules

Workflow:
- DRAFT → IN_REVIEW → PUBLISHED → ARCHIVED

Publishing:
- Creates immutable `SolutionVersion`
- Only one active version per Solution
- Published versions cannot be modified

All admin actions must create `AuditLog`.

---

## 6) SSR Rules

- Generated from template stored on `SolutionVersion`
- Deterministic
- Must include solution ID + version number
- Represents configuration, not installation

---

## 7) Scenario Handling

Each `scenario_type` must have:
- Pydantic input schema
- Normalization to `vars`
- Unit normalization (mm default)
- Optional `site_measurements` namespace (future validation)

Supported types:
- PENETRATION_SINGLE
- PENETRATION_MIXED
- VENT_DUCT
- LINEAR_JOINT
- BOARD_ENCASEMENT

---

## 8) Evaluation Flow (Must Follow Exactly)

1. Validate input via Pydantic
2. Normalize to `vars`
3. Fetch candidate SolutionVersions (active only)
4. Filter by applicability
5. Compute outputs via DSL
6. Evaluate constraints
7. Generate SSR
8. Return structured solution cards

Never mix steps.

---

## 9) Security Requirements

- JWT access + refresh tokens
- RBAC for admin routes
- Rate limiting for evaluate endpoint
- Signed S3 URLs only
- No stack traces in production

---

## 10) Testing Requirements

### Unit Tests
- DSL evaluator
- case operator
- lookup operator
- applicability matching
- constraint evaluation
- SSR generation

### Integration Tests
- Evaluate endpoint
- Solution detail endpoint
- Export queue job (mock allowed)

CI must pass before merge.

---

## 11) Seed Data Requirements

Seed script must include:
- 1 SUPER_ADMIN
- 1–2 manufacturers
- 2–3 products
- At least 1 published SolutionVersion with full rules
- 2 scenario examples

System must be testable locally after seed.

---

## 12) Export Rules

- HTML template rendered via Playwright
- Stored in S3
- Return signed URL
- Must use queue (no blocking export)

Use idempotency key:
hash(solutionVersionId + scenario_payload + template_version)

---

## 13) Error Handling Contract

All errors must return:

{
  "code": "ERROR_CODE",
  "message": "Human readable message",
  "details": {}
}

No internal stack traces in response.

---

## 14) Performance Constraints

- Avoid N+1 queries
- Load minimal relations required
- Applicability filtering should happen in DB where possible
- Use JSONB indexes if needed

---

## 15) Logging Standards

- Structured logs
- Include request_id
- Log:
  - evaluate start
  - solution count
  - export job creation
  - publishing actions

No sensitive data in logs.

---

## 16) Definition of Done (DoD)

Feature is complete only if:
- Migrations apply cleanly
- Seed data works
- Endpoints return correct structure
- Tests pass
- OpenAPI accurate
- No rule logic in frontend

---

## 17) Do NOT Build (Scope Protection)

The following are explicitly out of scope:

- Manufacturer dashboards
- AI solution ranking
- Installation certification workflows
- Inspector portals
- BIM/AR integration
- AR visualization engines
- CAD parsing
- Compliance authority claims

DIS backend = deterministic compliance intelligence engine only.

---

## 18) Architectural Discipline

If unsure:
- Keep logic in domain layer.
- Keep DB access in repository layer.
- Keep routers thin.
- Never mutate published data.
- Never weaken rule engine safety.

---

## 19) Long-Term Vision Awareness (But Not Implementation)

This backend is foundation for:
- Digital Installation Standard
- Industry reference credibility
- Cross-manufacturer compliance engine
- Future validation tools

But current implementation must remain deterministic, structured, and auditable.

Build foundation. Not features.