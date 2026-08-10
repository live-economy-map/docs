## Project: Shadow Economy Map — API Specification
**Feature:** Growth Map & Exploration
**Conventions:** see 0.1 in `00-api-conventions.md` — base path `/api/v1`, Public auth (no token) throughout this file, standard envelope, ApiError/validate middleware.

---

### 1.1 Endpoint Table

| Method | Path | Auth | Linked Use Case | Linked FR |
|---|---|---|---|---|
| GET | /map/cells | Public | UC-01, UC-02 | FR-01, FR-02, FR-10, FR-28 |
| GET | /map/cells/:cellId | Public | UC-03 | FR-03, FR-04 |
| GET | /map/layers/:sourceKey | Public | UC-04 | FR-05 |
| POST | /map/search | Public | UC-10 | FR-14 |

---

### 1.2 Endpoint Detail

#### GET /map/cells

**Purpose:** Return the composite growth-score layer for every grid cell, for a given (or default) time period. Powers the initial map load and every time-slider move.

**Auth:** Public

**Query params:**
```
?period=2026-06-01
```
`period` optional — an ISO date representing the target month. If omitted, or if no data exists for the requested period, the system falls back to the most recent available period (FR-10) and indicates the substitution in the response (see below) rather than erroring.

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "period": "2026-06-01",
    "periodSubstituted": false,
    "cells": [
      {
        "cellId": "uuid",
        "cellRow": 12,
        "cellCol": 8,
        "boundaryGeoJson": { "type": "Polygon", "coordinates": [] },
        "compositeScore": 0.74,
        "isComplete": true
      }
    ]
  }
}
```
`periodSubstituted: true` indicates the requested period had no data and the nearest available period was returned instead (per UC-02's alternate flow) — `period` reflects the period actually returned, not the one requested.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | Zod validation failure (e.g., `period` not a valid ISO date) | field-specific |

A period with zero data anywhere (no fallback available either) is not an error — it returns `cells: []` with `periodSubstituted: false` and an empty result, per the "non-error edge cases" convention.

**Implemented in:** `src/controllers/map.controller.ts → getCells` · `src/services/map.service.ts → getCompositeScoreLayer` · `src/schemas/map.schema.ts → getCellsQuerySchema`

---

#### GET /map/cells/:cellId

**Purpose:** Full click-to-inspect detail for a single grid cell — composite score, raw signal breakdown, trend, sparkline, AI summary. Powers UC-03.

**Auth:** Public

**Query params:**
```
?period=2026-06-01
```
Same optional/fallback behavior as `GET /map/cells`.

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "cellId": "uuid",
    "period": "2026-06-01",
    "areaLabel": "CMC",
    "compositeScore": 0.74,
    "isComplete": true,
    "trend": "up",
    "sparkline": [
      { "period": "2026-03-01", "compositeScore": 0.61 },
      { "period": "2026-04-01", "compositeScore": 0.66 },
      { "period": "2026-05-01", "compositeScore": 0.70 },
      { "period": "2026-06-01", "compositeScore": 0.74 }
    ],
    "signals": [
      { "source": "VIIRS", "rawValue": 12.4, "normalizedValue": 0.68 },
      { "source": "GHSL", "rawValue": 0.31, "normalizedValue": 0.55 },
      { "source": "RWI", "rawValue": 0.42, "normalizedValue": 0.60 }
    ],
    "aiSummary": "This area shows rising construction activity with stable night-light levels — consistent with early-stage development.",
    "lastUpdated": "2026-08-01T00:00:00Z"
  }
}
```
`trend` is one of `up | down | flat`, computed by comparing `compositeScore` to the immediately preceding period's snapshot. `aiSummary` may arrive slightly after the rest of the payload from the client's perspective if generated on-demand (see NFR on AI summary latency in Doc 2) — if generation fails, `aiSummary` is `null` rather than blocking the response (per UC-03's alternate flow).

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | Zod validation failure | field-specific |
| 404 | `cellId` doesn't exist, or has no data for the resolved period at all | "No data available for this cell" |

**Implemented in:** `src/controllers/map.controller.ts → getCellDetail` · `src/services/map.service.ts → getCellDetail, generateCellSummary` · `src/schemas/map.schema.ts → getCellDetailParamsSchema`

---

#### GET /map/layers/:sourceKey

**Purpose:** Return a single raw data layer (not the composite score) for the map's layer-toggle feature (UC-04).

**Auth:** Public

**Path params:** `sourceKey` — one of `VIIRS | GHSL | RWI`. `GDELT` is intentionally not a valid value here (see Doc 2 FR-05 note) — requesting it returns a 400, not an empty layer, since it's a client error (GDELT is exposed only via the Case Studies feature, not as a map layer).

**Query params:**
```
?period=2026-06-01
```
Same optional/fallback behavior as above.

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "sourceKey": "VIIRS",
    "period": "2026-06-01",
    "periodSubstituted": false,
    "cells": [
      { "cellId": "uuid", "normalizedValue": 0.68 }
    ]
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | `sourceKey` is not one of `VIIRS \| GHSL \| RWI` (including `GDELT`) | "Invalid layer — must be one of VIIRS, GHSL, RWI" |
| 400 | Zod validation failure on `period` | field-specific |

A layer with no data for the resolved period returns `cells: []`, not an error, per convention.

**Implemented in:** `src/controllers/map.controller.ts → getRawLayer` · `src/services/map.service.ts → getRawSignalLayer` · `src/schemas/map.schema.ts → getRawLayerParamsSchema`

---

#### POST /map/search

**Purpose:** Parse a natural-language query into map filters (location, time range, signal type) and return matching cells (UC-10).

**Auth:** Public

**Request body:**
```json
{
  "query": "string, required, 3–200 chars — e.g. 'show me areas near Bole with rising construction'"
}
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "parsedFilters": {
      "areaLabel": "Bole",
      "period": "2026-06-01",
      "signalFocus": "GHSL"
    },
    "cells": [
      { "cellId": "uuid", "compositeScore": 0.74 }
    ]
  }
}
```

**Success response, query not confidently parsed — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "Couldn't understand that query — try rephrasing, e.g. 'areas near Bole with rising construction'",
  "data": {
    "parsedFilters": null,
    "cells": []
  }
}
```
Per UC-10's alternate flow, a query the AI cannot confidently parse into valid filters is **not an error** — it's a normal `200` with `parsedFilters: null`, leaving the current map view unchanged on the client rather than guessing at intent.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | Zod validation failure (e.g., `query` empty or over 200 chars) | field-specific |
| 400 | AI service is unreachable/times out (distinct from "couldn't parse" above) | "Search is temporarily unavailable" |

**Implemented in:** `src/controllers/map.controller.ts → searchMap` · `src/services/map.service.ts → parseNaturalLanguageQuery` · `src/schemas/map.schema.ts → searchMapBodySchema`

---

**Next:** proceed to → [02. Case Studies & Validation API]