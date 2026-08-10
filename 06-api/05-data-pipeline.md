## Project: Shadow Economy Map — API Specification
**Feature:** Data Pipeline Management
**Conventions:** see 0.1 in `00-api-conventions.md` — base path `/api/v1`, Admin auth (Bearer JWT) throughout this file, standard envelope, ApiError/validate middleware.

---

### 5.1 Endpoint Table

| Method | Path | Auth | Linked Use Case | Linked FR |
|---|---|---|---|---|
| GET | /admin/pipeline/sources | Admin | UC-12 | FR-19, FR-20 |
| POST | /admin/pipeline/sources/:sourceKey/refresh | Admin | UC-12 | FR-18 |
| GET | /admin/pipeline/runs | Admin | UC-12 | FR-24 |
| POST | /admin/pipeline/recompute | Admin | UC-13 | FR-21 |
| GET | /admin/pipeline/weight-configs | Admin | UC-13 | FR-22 |
| POST | /admin/pipeline/weight-configs | Admin | UC-13 | FR-22 |

---

### 5.2 Endpoint Detail

#### GET /admin/pipeline/sources

**Purpose:** List all four data sources with their current health status and last successful update — powers the pipeline dashboard's overview (UC-12).

**Auth:** Admin

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "sources": [
      {
        "key": "VIIRS",
        "name": "VIIRS Night-Time Lights",
        "isActive": true,
        "lastSuccessfulRunAt": "2026-08-01T02:00:00Z",
        "healthStatus": "healthy"
      },
      {
        "key": "GDELT",
        "name": "GDELT",
        "isActive": true,
        "lastSuccessfulRunAt": null,
        "healthStatus": "stale"
      }
    ]
  }
}
```
`healthStatus` is one of `healthy | stale | failed | never_run`, computed server-side from the most recent `PipelineRun` per source (per Doc 4's design — not stored redundantly on `DataSource`).

**Error responses:** none beyond the common statuses in 0.1 (401 if unauthenticated).

**Implemented in:** `src/controllers/adminPipeline.controller.ts → listSources` · `src/services/adminPipeline.service.ts → getSourcesWithHealth`

---

#### POST /admin/pipeline/sources/:sourceKey/refresh

**Purpose:** Trigger a data pull for a single source, for the Addis Ababa bounding box (UC-12).

**Auth:** Admin

**Path params:** `sourceKey` — one of `VIIRS | GHSL | RWI | GDELT`.

**Success response — 202:**
```json
{
  "statusCode": 202,
  "success": true,
  "message": "Refresh started",
  "data": {
    "pipelineRunId": "uuid",
    "status": "RUNNING"
  }
}
```
Returns `202 Accepted`, not `200`, since the pull runs asynchronously — the client polls `GET /admin/pipeline/runs` or re-fetches `GET /admin/pipeline/sources` to see the result, rather than blocking on a potentially slow external API call.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | `sourceKey` not a valid `DataSourceKey` | "Invalid data source" |
| 409 | A run for this source is already `RUNNING` | "A refresh for this source is already in progress" |

A failed pull itself is not an error response from this endpoint — the request succeeded in *starting* the run; the failure is reflected asynchronously in the `PipelineRun` record (per Doc 3 UC-12's alternate flow: failure marks status and logs it, leaving prior data untouched).

**Implemented in:** `src/controllers/adminPipeline.controller.ts → triggerRefresh` · `src/services/adminPipeline.service.ts → startPipelineRun` · `src/schemas/adminPipeline.schema.ts → refreshParamsSchema`

---

#### GET /admin/pipeline/runs

**Purpose:** List past pipeline runs for observability (UC-12).

**Auth:** Admin

**Query params:**
```
?sourceKey=VIIRS&page=1&limit=20
```
`sourceKey` optional filter; pagination per convention.

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
        "dataSourceKey": "VIIRS",
        "status": "SUCCESS",
        "startedAt": "2026-08-01T02:00:00Z",
        "completedAt": "2026-08-01T02:04:12Z",
        "recordsProcessed": 340,
        "errorMessage": null
      }
    ],
    "page": 1,
    "limit": 20,
    "total": 57
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | Zod validation failure | field-specific |

**Implemented in:** `src/controllers/adminPipeline.controller.ts → listRuns` · `src/services/adminPipeline.service.ts → getPipelineRuns`

---

#### POST /admin/pipeline/recompute

**Purpose:** Trigger recomputation of the composite score for a given period, using the currently active weight config (UC-13).

**Auth:** Admin

**Request body:**
```json
{
  "period": "string (ISO date), required"
}
```

**Success response — 202:**
```json
{
  "statusCode": 202,
  "success": true,
  "message": "Recomputation started",
  "data": {
    "period": "2026-06-01",
    "scoreWeightConfigId": "uuid"
  }
}
```
Asynchronous for the same reason as the refresh endpoint above — recomputing across every grid cell for a period may take longer than a typical request timeout.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | Zod validation failure | field-specific |
| 409 | No `ScoreWeightConfig` is currently active | "No active weight configuration — create one first" |

Cells with missing source data during recomputation are not an error at this level — they're individually flagged `isComplete: false` on their `CompositeScoreSnapshot` row (per Doc 3 UC-13's alternate flow), while the overall recompute request still succeeds.

**Implemented in:** `src/controllers/adminPipeline.controller.ts → triggerRecompute` · `src/services/adminPipeline.service.ts → recomputeCompositeScores` · `src/schemas/adminPipeline.schema.ts → recomputeBodySchema`

---

#### GET /admin/pipeline/weight-configs

**Purpose:** List weight configurations, most recent first, to review history or find the currently active one (UC-13).

**Auth:** Admin

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "configs": [
      {
        "id": "uuid",
        "isActive": true,
        "createdAt": "2026-07-15T00:00:00Z",
        "weights": [
          { "sourceKey": "VIIRS", "weight": 0.4 },
          { "sourceKey": "GHSL", "weight": 0.35 },
          { "sourceKey": "RWI", "weight": 0.25 }
        ]
      }
    ]
  }
}
```

**Error responses:** none beyond the common statuses in 0.1.

**Implemented in:** `src/controllers/adminPipeline.controller.ts → listWeightConfigs` · `src/services/adminPipeline.service.ts → getWeightConfigs`

---

#### POST /admin/pipeline/weight-configs

**Purpose:** Create a new weight configuration (and mark it active), for experimentation/tuning (UC-13).

**Auth:** Admin

**Request body:**
```json
{
  "weights": [
    { "sourceKey": "VIIRS", "weight": 0.4 },
    { "sourceKey": "GHSL", "weight": 0.35 },
    { "sourceKey": "RWI", "weight": 0.25 }
  ]
}
```
Must include exactly the three scoreable sources (`VIIRS`, `GHSL`, `RWI`) — `GDELT` is rejected here, consistent with it never having a `SourceWeight` row (per Doc 4). Whether weights must sum to 1.0 is enforced here at the application layer — flagged as an open detail in Doc 4, to be confirmed before this validation is finalized.

**Success response — 201:**
```json
{
  "statusCode": 201,
  "success": true,
  "message": "Weight configuration created and activated",
  "data": {
    "id": "uuid",
    "isActive": true,
    "weights": [
      { "sourceKey": "VIIRS", "weight": 0.4 },
      { "sourceKey": "GHSL", "weight": 0.35 },
      { "sourceKey": "RWI", "weight": 0.25 }
    ]
  }
}
```
Creating a new config automatically deactivates the previously active one (only one `isActive: true` at a time, per Doc 4).

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | Zod validation failure (missing source, duplicate source, weight out of range) | field-specific |
| 400 | Weights don't sum to 1.0 *(pending confirmation — see note above; may not apply if a different formula is chosen)* | "Weights must sum to 1.0" |
| 400 | `GDELT` included in `weights` | "GDELT cannot be assigned a weight" |

**Implemented in:** `src/controllers/adminPipeline.controller.ts → createWeightConfig` · `src/services/adminPipeline.service.ts → createAndActivateWeightConfig` · `src/schemas/adminPipeline.schema.ts → createWeightConfigSchema`

---

**Next:** proceed to → [06. Case Study Curation API]