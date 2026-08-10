## Project: Shadow Economy Map — API Specification
**Feature:** Case Study Curation
**Conventions:** see 0.1 in `00-api-conventions.md` — base path `/api/v1`, Admin auth (Bearer JWT) throughout this file, standard envelope, ApiError/validate middleware.

---

### 6.1 Endpoint Table

| Method | Path | Auth | Linked Use Case | Linked FR |
|---|---|---|---|---|
| GET | /admin/case-studies | Admin | UC-14 | FR-23 |
| POST | /admin/case-studies | Admin | UC-14 | FR-23 |
| PATCH | /admin/case-studies/:caseStudyId | Admin | UC-14 | FR-23 |
| DELETE | /admin/case-studies/:caseStudyId | Admin | UC-14 | FR-23 |
| POST | /admin/case-studies/discover | Admin | UC-15 | FR-25 |

Note the deliberate separation from the public `02-case-studies-api.md` — same `CaseStudy` entity, but this file's endpoints see **all** case studies (published and draft), while the public file only ever sees `isPublished: true`.

---

### 6.2 Endpoint Detail

#### GET /admin/case-studies

**Purpose:** List all case studies, published and draft, for admin management (UC-14).

**Auth:** Admin

**Query params:**
```
?isPublished=false&page=1&limit=20
```
`isPublished` optional filter (unlike the public endpoint, this one can return drafts); pagination per convention.

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
        "name": "Gerji development zone",
        "isPublished": false,
        "evidenceTier": "LOCAL_NEWS",
        "createdAt": "2026-08-05T00:00:00Z"
      }
    ],
    "page": 1,
    "limit": 20,
    "total": 5
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | Zod validation failure | field-specific |

**Implemented in:** `src/controllers/adminCaseStudies.controller.ts → listAllCaseStudies` · `src/services/adminCaseStudies.service.ts → getAllCaseStudies`

---

#### POST /admin/case-studies

**Purpose:** Create a new case study (UC-14). Created as a draft (`isPublished: false`) unless explicitly published.

**Auth:** Admin

**Request body:**
```json
{
  "name": "string, required",
  "latitude": "number, required",
  "longitude": "number, required",
  "gridCellId": "string (uuid), optional",
  "evidenceDescription": "string, required",
  "evidenceUrl": "string (url), optional",
  "evidenceTier": "OFFICIAL | MARKET_REPORT | INFRASTRUCTURE | LOCAL_NEWS, optional",
  "scoreRiseDate": "string (ISO date), required",
  "confirmedDate": "string (ISO date), required",
  "beforeImageUrl": "string (url), optional",
  "afterImageUrl": "string (url), optional",
  "isPublished": "boolean, optional — defaults to false"
}
```

**Success response — 201:**
```json
{
  "statusCode": 201,
  "success": true,
  "message": "Case study created",
  "data": {
    "id": "uuid",
    "name": "Gerji development zone",
    "isPublished": false
  }
}
```

**Success response, dates out of expected order — 201:**
```json
{
  "statusCode": 201,
  "success": true,
  "message": "Case study created — note: scoreRiseDate is after confirmedDate, which is unexpected for a validation case study",
  "data": {
    "id": "uuid",
    "name": "Gerji development zone",
    "isPublished": false
  }
}
```
Per the Phase 2/4 discussion establishing `scoreRiseDate <= confirmedDate` as the expected validation pattern: this is **not rejected as an error**, since an admin may be entering exploratory or intentionally-flagged data — but it's surfaced as a warning message so it isn't published unnoticed. The frontend should visually flag this case for review before it's published.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | Zod validation failure (missing required fields, invalid URLs, invalid dates) | field-specific |
| 404 | `gridCellId` provided but doesn't exist | "Grid cell not found" |

**Implemented in:** `src/controllers/adminCaseStudies.controller.ts → createCaseStudy` · `src/services/adminCaseStudies.service.ts → createCaseStudy` · `src/schemas/adminCaseStudies.schema.ts → createCaseStudySchema`

---

#### PATCH /admin/case-studies/:caseStudyId

**Purpose:** Update an existing case study, including publishing it (setting `isPublished: true`) (UC-14).

**Auth:** Admin

**Request body:** any subset of the fields listed under `POST` above.

**Success response — 200:** same shape as the create response, reflecting updated fields.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | Zod validation failure | field-specific |
| 404 | `caseStudyId` doesn't exist | "Case study not found" |

**Implemented in:** `src/controllers/adminCaseStudies.controller.ts → updateCaseStudy` · `src/services/adminCaseStudies.service.ts → updateCaseStudy` · `src/schemas/adminCaseStudies.schema.ts → updateCaseStudySchema`

---

#### DELETE /admin/case-studies/:caseStudyId

**Purpose:** Remove a case study entirely (UC-14).

**Auth:** Admin

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "Case study deleted",
  "data": {}
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 404 | `caseStudyId` doesn't exist | "Case study not found" |

**Implemented in:** `src/controllers/adminCaseStudies.controller.ts → deleteCaseStudy` · `src/services/adminCaseStudies.service.ts → deleteCaseStudy`

---

#### POST /admin/case-studies/discover

**Purpose:** Internal AI-assisted search to surface case-study candidates from news sources, for manual verification (UC-15, FR-25). Not a public-facing feature.

**Auth:** Admin

**Request body:**
```json
{
  "areaFocus": "string, required — e.g. 'Ayat' or 'Lemi Kura'"
}
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "candidates": [
      {
        "summary": "string — brief AI-generated summary of a relevant news mention",
        "sourceUrl": "https://example.com/article",
        "suggestedEvidenceTier": "LOCAL_NEWS",
        "mentionedDate": "2026-07-20"
      }
    ]
  }
}
```
Every candidate is explicitly a **suggestion requiring manual verification** — this endpoint never auto-creates a `CaseStudy` row. An admin reviews the results and, if verified, manually calls `POST /admin/case-studies` to actually create one (per UC-15's postcondition).

**Success response, no candidates found — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "No relevant candidates found",
  "data": { "candidates": [] }
}
```
Per UC-15's alternate flow: no results is a normal empty response, never a fabricated plausible-sounding suggestion.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | Zod validation failure (`areaFocus` empty) | field-specific |
| 400 | AI/search service unreachable or times out | "Discovery search is temporarily unavailable" |

**Implemented in:** `src/controllers/adminCaseStudies.controller.ts → discoverCandidates` · `src/services/adminCaseStudies.service.ts → searchCaseStudyCandidates` · `src/schemas/adminCaseStudies.schema.ts → discoverBodySchema`

---

**Next:** proceed to → [System Architecture]