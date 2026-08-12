## Project: Shadow Economy Map — API Specification
**Feature:** Site Content & Onboarding
**Conventions:** see 0.1 in `00-api-conventions.md` — base path `/api/v1`, Public auth (no token) throughout this file, standard envelope, ApiError/validate middleware.

---

### 3.1 Endpoint Table

| Method | Path | Auth | Linked Use Case | Linked FR |
|---|---|---|---|---|
| GET | /content/landing | Public | UC-07 | FR-11 |
| GET | /content/methodology | Public | UC-06 | FR-09 |

The map legend/onboarding overlay (UC-09, FR-13) is intentionally **not** listed here — its content is static copy shipped with the frontend rather than served from the API, since it never changes independently of a deploy. If that changes later (e.g., admin-editable onboarding copy), it would move here as a new endpoint at that point, per the "append, don't restructure" numbering discipline established in Doc 3.

---

### 3.2 Endpoint Detail

#### GET /content/landing

**Purpose:** Content for the landing/home page — introductory copy and any highlighted stats shown before a visitor enters the map (UC-07).

**Auth:** Public

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "tagline": "See the economy that no one measures.",
    "intro": "string — short introductory copy",
    "highlightStats": {
      "publishedCaseStudyCount": 3,
      "lastDataRefresh": "2026-08-01T00:00:00Z"
    }
  }
}
```
`highlightStats` gives the landing page a small amount of live, real data (e.g., "3 validated case studies," "data current as of August 2026") without needing a separate call — computed server-side from `CaseStudy` and `PipelineRun` at request time.

**Error responses:** none beyond the common statuses in 0.1 (no input, nothing to validate).

**Implemented in:** `src/controllers/content.controller.ts → getLandingContent` · `src/services/content.service.ts → getLandingStats`

---

#### GET /content/methodology

**Purpose:** Content for the About/Methodology page — explanation of the composite score, the four data sources, how case studies are validated, and known limitations (UC-06).

**Auth:** Public

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "scoreExplanation": "string — plain-language explanation of the composite score",
    "dataSources": [
      {
        "key": "VIIRS",
        "name": "VIIRS Night-Time Lights",
        "description": "string"
      },
      {
        "key": "GHSL",
        "name": "Global Human Settlement Layer",
        "description": "string"
      },
      {
        "key": "RWI",
        "name": "Meta Relative Wealth Index",
        "description": "string"
      },
      {
        "key": "GDELT",
        "name": "GDELT",
        "description": "string — explains its validation-only role, not a scored input"
      }
    ],
    "validationApproach": "string — explains the tiered case-study sourcing and score-rise-vs-confirmation logic",
    "limitations": [
      "Detects current/recent activity only — does not forecast future growth",
      "Covers Addis Ababa only",
      "Does not measure informal/tax-related economic activity specifically",
      "Some inputs (RWI) are coarse resolution (~2.4km)",
      "Map data reflects the last admin-triggered refresh, not real-time"
    ]
  }
}
```
`dataSources` is returned dynamically from the `DataSource` table (per Doc 4's design intent — a table, not hardcoded), so the four source descriptions stay in sync with whatever admins configure, without a frontend redeploy.

**Error responses:** none beyond the common statuses in 0.1.

**Implemented in:** `src/controllers/content.controller.ts → getMethodologyContent` · `src/services/content.service.ts → getMethodologyContent`

---

**Next:** proceed to → [04. Admin Access API]