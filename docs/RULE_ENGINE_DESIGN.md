# Rule Engine Design

## Core Concept

Scenario-based deterministic evaluation.

Each solution belongs to a scenario_type.

---

## Supported Scenario Types

- PENETRATION_SINGLE
- PENETRATION_MIXED
- VENT_DUCT
- LINEAR_JOINT
- BOARD_ENCASEMENT

Each scenario_type defines:
- Required input schema
- Validation rules
- Computation pipeline

---

## Evaluation Pipeline

1. Validate scenario input
2. Fetch candidate solutions by scenario_type
3. Filter by applicability constraints
4. Apply computation rules
5. Generate derived requirements
6. Return solution cards
7. Generate SSR for selected solution

---

## SSR Generation

SSR Format:
SSR-{DISCIPLINE}-{TYPE}-{SIZE}-{FIRE}-{CONSTRUCTION}-{MFR}-{SOL}-{VERSION}

Example:
SSR-FP-VENT-315-EI60-MW-OEL-S01-V2

SSR identifies solution configuration, not installation event.