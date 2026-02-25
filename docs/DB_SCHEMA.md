# Database Schema Overview

## manufacturers
- id (PK)
- name
- description
- status
- created_at

## products
- id (PK)
- manufacturer_id (FK)
- name
- category
- technical_specs (JSONB)

## solutions
- id (PK)
- manufacturer_id (FK)
- title
- discipline (FIRE)
- scenario_type
- shape (ROUND, RECTANGULAR, MIXED, JOINT, BOARD)
- status (DRAFT, REVIEW, PUBLISHED, ARCHIVED)
- created_at

## solution_versions
- id (PK)
- solution_id (FK)
- version_number
- published_at
- is_active
- ssr_template

## solution_applicability
- id
- solution_version_id (FK)
- construction_types (JSONB)
- fire_ratings (JSONB)
- service_types (JSONB)
- diameter_min
- diameter_max
- width_min
- width_max
- height_min
- height_max
- installation_sides (JSONB)

## solution_rules
- id
- solution_version_id (FK)
- hole_formula
- tolerance_rule
- clearance_rule
- insulation_rule
- custom_constraints (JSONB)

## solution_materials
- id
- solution_version_id (FK)
- product_id (FK optional)
- description
- quantity_logic (JSONB)

## solution_steps
- id
- solution_version_id (FK)
- step_number
- instruction_text

## solution_references
- id
- solution_version_id (FK)
- reference_code
- document_url

## audit_logs
- id
- user_id
- action
- entity_type
- entity_id
- timestamp