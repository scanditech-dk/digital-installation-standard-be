# BACKEND.md — ScandiTech Digital Installation Standard (DIS)
## Fire Protection Edition — Backend Architecture & Build Guide (Authoritative)

This document is the single source of truth for backend engineering decisions, patterns, and implementation rules for DIS.

---

## 1) Purpose

DIS backend provides:
- Deterministic evaluation of fire protection scenarios
- Multi-manufacturer compliant solution listing (participating manufacturers only)
- Standardized solution output (SSR + requirements + steps + materials + references)
- ScandiTech-only content publishing workflow (no manufacturer portal)
- Exportable Solution Pack (PDF)

DIS backend does **not** certify installation execution.

---

## 2) Tech Stack (Locked)

**Runtime**
- Node.js + TypeScript

**Framework**
- NestJS

**Database**
- PostgreSQL
- Prisma

**Queue**
- Redis + BullMQ

**File Storage**
- S3-compatible storage (AWS S3 / MinIO / etc.)

**PDF Generation**
- Playwright (HTML → PDF)

**Validation**
- Zod (preferred) or class-validator (acceptable)

**Docs**
- OpenAPI/Swagger via NestJS

**Logging/Monitoring**
- Pino (structured logs)
- Sentry (errors) optional but recommended

---

## 3) Core Domain Model (Publisher-Controlled)

### Entities (core)
- Manufacturer (data entity only)
- Product (optional but recommended)
- Solution (draft container)
- SolutionVersion (immutable published snapshot)
- Applicability (filters)
- Rules (compute + constraints DSL)
- Steps (ordered instructions)
- Materials (BOM + quantity logic)
- References (approvals/test documents)
- AuditLog (who changed what)

### Important Rules
- ScandiTech is the only content publisher.
- Manufacturers have **no login** and no dashboard.
- Published solution versions are immutable.
- Any change creates a new `SolutionVersion`.

---

## 4) Module Boundaries (NestJS)

Recommended top-level modules:

- `AuthModule`
- `UsersModule`
- `OrganizationsModule` (multi-tenant customers)
- `ManufacturersModule`
- `ProductsModule`
- `SolutionsModule`
  - Draft workflows
  - Version publishing
  - Retrieval
- `ScenarioModule`
  - Scenario schemas (by scenario_type)
  - Normalization
- `RuleEngineModule`
  - Candidate selection (applicability filtering)
  - Computation
  - Constraints evaluation
  - SSR generation
- `ExportModule`
  - HTML templates
  - PDF generation worker job
- `FilesModule` (S3)
- `AuditModule`

---

## 5) Rule Engine (The Brain)

### Responsibilities
Given a validated scenario input:
1. Normalize inputs → typed `vars`
2. Fetch candidate `SolutionVersion` records for `scenario_type`
3. Apply applicability filtering (fire rating, construction, shape, size range, etc.)
4. Compute derived outputs using safe DSL (no eval)
5. Evaluate constraints (warnings/errors)
6. Return solution cards with computed requirements + SSR preview

### Non-Negotiables
- Deterministic (no AI selection in core evaluation)
- Safe expression engine (whitelisted operators only)
- Explainable (optional debug output for admin mode)
- Versioned output (include `engineVersion` and `solutionVersionId`)

---

## 6) Scenario Types (Supported from Day 1)

Scenario types are part of system design even if content grows over time:

- `PENETRATION_SINGLE`
- `PENETRATION_MIXED`
- `VENT_DUCT`
- `LINEAR_JOINT`
- `BOARD_ENCASEMENT`

Each scenario type must have:
- Input schema definition
- Normalization mapping to `vars`
- Dimension model support (round/rectangular)
- Optional “site_measurements” namespace (for future validation)

---

## 7) DSL Rule JSON (Safe Compute + Constraints)

Rules are stored as structured JSON (no code strings):
- `computed.outputs.{field}.compute` uses AST operators (`add`, `mul`, `case`, `lookup`, etc.)
- `constraints[]` includes:
  - `when` boolean expression
  - `assert` boolean expression
  - `severity` (WARN/ERROR)
  - user-facing `message`

Implementation MUST NOT support `eval` or arbitrary code execution.

---

## 8) SSR (Standard Solution Reference)

### Goal
Generate stable solution-level reference codes for documentation.

### Behavior
- SSR is generated at evaluation time for each returned solution card.
- SSR identifies the selected solution version + key scenario variables.
- SSR does not represent installation certification.

### Example Template
`SSR-FP-VENT-{SHAPE}-{DIAMETER}-{FIRE}-{CONSTRUCTION}-{MFR}-S{SOLUTION}-V{VER}`

---

## 9) API Surface (Minimum)

### Public (Customer) APIs
- `POST /v1/evaluate`
  - input: scenario payload
  - output: list of compliant solutions with computed requirements + SSR

- `GET /v1/solutions/:solutionVersionId`
  - output: solution pack data (steps/materials/references + computed requirements)

- `POST /v1/exports/solution-pack`
  - input: `solutionVersionId` + scenario payload (for computed values)
  - output: export job id or direct PDF link

### Admin (ScandiTech) APIs
- Manufacturer CRUD
- Product CRUD
- Solution draft CRUD
- Publish workflow:
  - `POST /v1/admin/solutions/:solutionId/publish`
    - creates immutable `SolutionVersion` snapshot

---

## 10) Publishing Workflow (ScandiTech Internal)

States:
- `DRAFT` → `IN_REVIEW` → `PUBLISHED` → `ARCHIVED`

Rules:
- Only reviewers/super_admin can publish
- Publishing creates a new immutable `SolutionVersion`
- Only one version active per `Solution`

Audit:
- Every create/edit/publish must write an `AuditLog`

---

## 11) Multi-Tenancy (Customers)

DIS supports customer organizations:
- `Organization` as tenant boundary
- `User.organizationId` for customer users
- Content (solutions) is global and shared
- Future: subscription plan gates access (not part of rule engine)

---

## 12) PDF Solution Pack Export

### Approach
- Generate HTML via server-side template (must include SSR + computed fields)
- Render with Playwright → PDF
- Store PDF in S3
- Return signed URL to client

### Queue Strategy
- `exports:solution-pack` BullMQ queue
- Retries for transient failures
- Idempotency key (hash of `solutionVersionId + scenario input + template version`)

---

## 13) Security & Compliance

- JWT auth + refresh tokens
- RBAC enforced via Nest guards
- Rate limiting for public endpoints
- Input validation required on every request
- Store all documents in private S3 buckets; serve via signed URLs

---

## 14) Testing Strategy

- Unit tests: DSL evaluator, applicability filter, SSR generator
- Integration tests: evaluate endpoint with seeded solution versions
- Migration tests: Prisma migrations in CI
- Use Testcontainers for Postgres/Redis in CI if possible

---

## 15) Developer Implementation Rules (Hard)

1. No eval of formulas.
2. Published solution versions are immutable.
3. Rule engine must be deterministic and explainable.
4. Admin edits must never mutate published versions.
5. All changes must be audited.
6. Keep compliance logic server-side only.

---

## 16) Immediate Deliverables (Backend Team)

1. Prisma schema + migrations
2. Auth + RBAC scaffold
3. Admin CRUD for:
   - manufacturers
   - products
   - solutions (draft)
   - publish to solution_versions
4. Rule engine MVP:
   - applicability filter
   - compute outputs via DSL
   - constraints evaluation
   - SSR generation
5. Evaluate endpoint + solution detail endpoint
6. Export service (Playwright + S3 + queue)

---

## 17) Environment Variables (Minimum)

- `DATABASE_URL`
- `REDIS_URL`
- `S3_ENDPOINT` (if not AWS)
- `S3_REGION`
- `S3_BUCKET`
- `S3_ACCESS_KEY_ID`
- `S3_SECRET_ACCESS_KEY`
- `JWT_ACCESS_SECRET`
- `JWT_REFRESH_SECRET`
- `PUBLIC_BASE_URL`
- `SENTRY_DSN` (optional)

---

## 18) What’s Next

After backend scaffold is live, we finalize:
- Scenario input JSON schemas per scenario_type
- Standardized output contract for frontend rendering
- Admin authoring UX requirements (fields, validations, preview)