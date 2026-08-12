## Project: Shadow Economy Map — API Specification
**Links back to:** [1. Problem & Solution Statement], [2. Functional & Non-Functional Requirements], [3. Use Cases], [4. Database & Data Model], [roadmap.md]

Split into feature-based files, mirroring the feature grouping used throughout the project (requirements, use cases, and later function-level specs):

- **00-api-conventions.md** — this file: shared rules referenced by every feature file below
- **01-growth-map-api.md** — Growth Map & Exploration
- **02-case-studies-api.md** — Case Studies & Validation
- **03-site-content-api.md** — Site Content & Onboarding
- **04-admin-access-api.md** — Admin Access
- **05-data-pipeline-api.md** — Data Pipeline Management
- **06-case-study-curation-api.md** — Case Study Curation

Each feature file has its own endpoint table + endpoint detail sections. This file is not repeated in each — feature files reference "see 0.1" etc. the way the source e-commerce template referenced its own conventions section.

---

### 0.1 Base Conventions

**Base path:** `/api/v1`

**Auth:**
- `Public` — no token required. Used by all Growth Map, Case Studies, and Site Content endpoints (V1 has no public user accounts).
- `Admin` — Bearer JWT required, via `Authorization: Bearer <token>`. `authMiddleware` attaches `req.user` (the authenticated `Admin` record). Used by all Admin Access, Data Pipeline Management, and Case Study Curation endpoints.

**Standard success envelope** (`SuccessResponse`):
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": { }
}
```

**Standard error envelope** (`ErrorResponse`, produced via `throw new ApiError(statusCode, message, errors[])`):
```json
{
  "statusCode": 400,
  "success": false,
  "message": "string",
  "errors": []
}
```

**Validation:** every endpoint accepting input is validated by a Zod schema via the `validate(schema)` middleware before the controller runs. Feature files show the Zod-shaped schema (`body` / `params` / `query`) per endpoint. A `400` with `errors: []` populated field-by-field is the standard shape for validation failures — not documented per-endpoint unless the endpoint has additional custom validation beyond schema shape (e.g., business-rule checks).

**Common error statuses**, not re-documented on every endpoint unless the condition is specific to that endpoint:
| Status | Meaning |
|---|---|
| 400 | Zod validation failure |
| 401 | Missing/invalid/expired admin token (Admin-auth endpoints only) |
| 404 | Resource not found |
| 500 | Unexpected server error (never intentionally thrown — see backend template's error flow) |

**Non-error edge cases:** as in the e-commerce template's pattern, an empty or substituted result is not an error. Examples specific to this project:
- No data for a requested time period → falls back to nearest available period, returned as a normal `200`, not a `404`
- A grid cell with incomplete source data → returned normally with `isComplete: false`, not omitted or errored
- A case-study or query with zero matches → returns an empty array, not a `404`

**Timestamps:** all `DateTime` fields are ISO 8601 strings (UTC) in both requests and responses, matching Doc 4's schema.

**Pagination** (where applicable, e.g., case studies list): `?page=1&limit=20` query params, defaulting to `page=1`, `limit=20`, following the same pattern as the e-commerce template's catalog endpoint.

**Implementation reference format:** every endpoint lists `Implemented in:` pointing to the exact controller → service → schema file paths, per the backend template's feature-folder pattern (`src/controllers/<feature>.controller.ts`, `src/services/<feature>.service.ts`, `src/schemas/<feature>.schema.ts`, `src/routes/<feature>.routes.ts`, registered in `src/routes/index.ts`).

---

### 0.2 Feature-to-Router Mapping

| Feature | Mounted at | Router file |
|---|---|---|
| Growth Map & Exploration | `/api/v1/map` | `src/routes/map.routes.ts` |
| Case Studies & Validation | `/api/v1/case-studies` | `src/routes/caseStudies.routes.ts` |
| Site Content & Onboarding | `/api/v1/content` | `src/routes/content.routes.ts` |
| Admin Access | `/api/v1/admin/auth` | `src/routes/adminAuth.routes.ts` |
| Data Pipeline Management | `/api/v1/admin/pipeline` | `src/routes/adminPipeline.routes.ts` |
| Case Study Curation | `/api/v1/admin/case-studies` | `src/routes/adminCaseStudies.routes.ts` |

Note the deliberate path split: public case-study viewing lives at `/case-studies`, while admin case-study management lives at `/admin/case-studies` — same underlying `CaseStudy` entity, two different routers/feature owners, consistent with the feature-ownership model discussed for team assignment.

---

**Next:** proceed to → [01. Growth Map & Exploration API]