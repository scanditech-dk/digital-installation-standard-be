# API Specification

## Public Endpoints

POST /v1/evaluate
Body:
{
  scenario_type,
  construction_type,
  fire_rating,
  service_type,
  shape,
  dimensions,
  installation_side
}

Returns:
{
  solutions: [],
  metadata
}

GET /v1/solutions/{solutionVersionId}

POST /v1/export/{solutionVersionId}

---

## Admin Endpoints

POST /v1/admin/manufacturers
POST /v1/admin/products
POST /v1/admin/solutions
PUT /v1/admin/solutions/{id}
POST /v1/admin/solutions/{id}/publish

---

## Auth & Roles

- super_admin
- content_admin
- reviewer
- customer_admin
- field_user