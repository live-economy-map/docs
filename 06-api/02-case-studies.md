## Project: Shadow Economy Map — API Specification
**Feature:** Case Studies & Validation
**Conventions:** see 0.1 in `00-api-conventions.md` — base path `/api/v1`, Public auth (no token) throughout this file, standard envelope, ApiError/validate middleware.

---

### 2.1 Endpoint Table

| Method | Path | Auth | Linked Use Case | Linked FR |
|---|---|---|---|---|
| GET | /case-studies | Public | UC-05, UC-08 | FR-06, FR-12 |
| GET | /case-studies/:caseStudyId | Public | UC-05, UC-08 | FR-06, FR-07, FR-08 |

Only `isPublished: true` case studies are ever returned by these endpoints — unpublished/draft entries are visible only via the Admin Case Study Curation feature (see `06-case-study-curation-api.md`).

---

### 2.2 Endpoint Detail

#### GET /case-studies

**Purpose:** List all published case studies — powers both the map's case-study markers (UC-05) and the standalone Case Studies list page (UC-08).

**Auth:** Public

**Query params:**
```
?page=1&limit=20
```
Optional, defaulting to `page=1`, `limit=20` per the pagination convention (0.1). In practice V1 expects only 2–4 published case studies, so pagination will rarely matter, but the shape stays consistent with the rest of the API.

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "items": [
      {
        "id": "uuid",
        "name": "CMC growth corridor",
        "latitude": 9.0192,
        "longitude": 38.8123,
        "scoreRiseDate": "2026-02-01",
        "confirmedDate": "2026-05-15",
        "evidenceTier": "MARKET_REPORT"
      }
    ],
    "page": 1,
    "limit": 20,
    "total": 3
  }
}
```
This is the summary shape (for map pins and list rows) — full evidence detail is fetched per-item via the detail endpoint below.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | Zod validation failure (e.g., `page`/`limit` not positive integers) | field-specific |

Zero published case studies is not an error — returns `items: []` (per UC-08's alternate flow: empty-state message is a client-side concern, not an API error).

**Implemented in:** `src/controllers/caseStudies.controller.ts → listCaseStudies` · `src/services/caseStudies.service.ts → getPublishedCaseStudies` · `src/schemas/caseStudies.schema.ts → listCaseStudiesQuerySchema`

---

#### GET /case-studies/:caseStudyId

**Purpose:** Full evidence detail for a single case study — supporting evidence, dates, and before/after imagery where available. Powers both clicking a map marker (UC-05) and selecting an item from the list page (UC-08).

**Auth:** Public — 404s if the case study exists but `isPublished: false`, identical to a non-existent case study (never reveals unpublished/draft entries publicly).

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "name": "CMC growth corridor",
    "latitude": 9.0192,
    "longitude": 38.8123,
    "evidenceDescription": "New residential and commercial construction confirmed by market analysis reports covering the CMC/Summit area.",
    "evidenceUrl": "https://example.com/report",
    "evidenceTier": "MARKET_REPORT",
    "scoreRiseDate": "2026-02-01",
    "confirmedDate": "2026-05-15",
    "beforeImageUrl": "https://example.com/before.jpg",
    "afterImageUrl": "https://example.com/after.jpg"
  }
}
```
`beforeImageUrl`/`afterImageUrl` are `null` when not available for this case study (per FR-08 being a "Should," not every case study is guaranteed imagery) — the client displays evidence and dates only in that case, without the comparison view (per UC-05's alternate flow).

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 404 | `caseStudyId` doesn't exist, or exists but `isPublished: false` | "Case study not found" |

**Implemented in:** `src/controllers/caseStudies.controller.ts → getCaseStudyById` · `src/services/caseStudies.service.ts → getPublishedCaseStudyById`

---

**Next:** proceed to → [03. Site Content & Onboarding API]