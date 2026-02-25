# System Architecture

## High-Level Components

1. Web Application (Customer)
2. Web Admin Application (ScandiTech)
3. Backend API
4. Rule Engine Module
5. PostgreSQL Database
6. File Storage (S3-compatible)
7. PDF Generation Service

---

## System Flow

User → Scenario Input → API → Rule Engine → DB → Compliant Solutions → SSR → Solution Pack

---

## Core Services

### Backend API
- Auth & RBAC
- Scenario evaluation
- Solution retrieval
- SSR generation
- Admin CRUD
- Version publishing
- Export generation

### Rule Engine
- Scenario validation
- Applicability filtering
- Computation rules
- Derived dimension calculation
- Structured output formatting

### Admin System
- Manufacturer management
- Product catalog
- Solution authoring
- Rule configuration
- Version publishing workflow

---

## Design Rules

- No compliance logic in frontend
- Rule logic separated from presentation
- Solutions immutable once published
- Version-based evaluation
- Audit logs for all content changes