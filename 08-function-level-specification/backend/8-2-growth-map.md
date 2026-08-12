## Project: Shadow Economy Map — Backend Function-Level Spec: Growth Map & Exploration
Files covered: `src/schemas/map.schema.ts` · `src/services/map.service.ts` · `src/controllers/map.controller.ts` · `src/routes/map.routes.ts`

---

### src/schemas/map.schema.ts (new)

| Schema | Shape |
|---|---|
| getCellsQuerySchema | `z.object({ query: z.object({ period: z.string().date().optional() }) })` |
| getCellDetailParamsSchema | `z.object({ params: z.object({ cellId: z.string().uuid() }), query: z.object({ period: z.string().date().optional() }) })` |
| getRawLayerParamsSchema | `z.object({ params: z.object({ sourceKey: z.enum(['VIIRS', 'GHSL', 'RWI']) }), query: z.object({ period: z.string().date().optional() }) })` |
| searchMapBodySchema | `z.object({ body: z.object({ query: z.string().min(3).max(200) }) })` |

`sourceKey` is validated as a 3-value enum, not the full 4-value `DataSourceKey` — `GDELT` is a schema-level 400 here, not a runtime check in the service, per API spec 1.2's explicit exclusion.

---

### src/services/map.service.ts (new)

#### getCompositeScoreLayer

| Field | Detail |
|---|---|
| Signature | `getCompositeScoreLayer(period?: string): Promise<{ period: string; periodSubstituted: boolean; cells: GrowthCellDTO[] }>` |
| Purpose | Returns every grid cell's latest composite score for the requested (or nearest available) period. |
| Inputs | `period` (ISO date, optional) |
| Output | Cells shape per API spec 1.2 |
| Throws | None |
| Side effects | Read-only `CompositeScoreSnapshot.findMany`, joined to `GridCell`, filtered to the active `ScoreWeightConfig`'s snapshots for the resolved period |
| Edge cases | No data for requested period → find nearest earlier period with any `CompositeScoreSnapshot` rows, set `periodSubstituted: true`. No data for any period → `cells: []`, `periodSubstituted: false`, not an error. |

Test file: `tests/services/map.service.test.ts`

#### getCellDetail

| Field | Detail |
|---|---|
| Signature | `getCellDetail(cellId: string, period?: string): Promise<CellDetailDTO>` |
| Purpose | Full click-to-inspect payload: composite score, raw signals, trend, sparkline, AI summary. |
| Inputs | `cellId`, `period` (optional) |
| Output | Shape per API spec 1.2 |
| Throws | `ApiError(404, "No data available for this cell")` — cell doesn't exist, or has zero `CompositeScoreSnapshot` rows across all periods |
| Side effects | Read-only queries against `GridCell`, `SignalValue` (3 sources), `CompositeScoreSnapshot` (current + prior period, for `trend`), and the last ~6 periods for `sparkline`. Calls `generateCellSummary` (below) for `aiSummary`. |
| Edge cases | `trend` requires a prior-period snapshot to exist — if none, return `trend: 'flat'` rather than throwing or guessing. If `generateCellSummary` fails/times out, `aiSummary: null` — the rest of the payload is still returned (per NFR on AI reliability, Doc 2). |

Test file: `tests/services/map.service.test.ts`

#### generateCellSummary

| Field | Detail |
|---|---|
| Signature | `generateCellSummary(signals: SignalValueDTO[], compositeScore: number): Promise<string | null>` |
| Purpose | Calls the LLM via `aiClient.ts` to produce a one-sentence plain-language summary of a cell's signals. |
| Inputs | The 3 raw signal values, the composite score |
| Output | A string, or `null` on failure |
| Throws | Never — internally catches all `aiClient` errors and returns `null` rather than propagating, since a summary failure must never block the rest of `getCellDetail`'s response (this is the one function in the whole backend explicitly designed to swallow its own errors, and that choice should not be "cleaned up" during implementation) |
| Side effects | External LLM API call |
| Edge cases | Prompt must explicitly instruct the model not to use forecasting language (e.g., "will grow," "is expected to") — enforced via the system prompt in `aiClient.ts`, not re-validated here; see NFR "AI reliability" in Doc 2. |

Test file: `tests/services/map.service.test.ts` — mock `aiClient`, cover both success and swallowed-failure paths.

#### getRawSignalLayer

| Field | Detail |
|---|---|
| Signature | `getRawSignalLayer(sourceKey: 'VIIRS' \| 'GHSL' \| 'RWI', period?: string): Promise<{ period: string; periodSubstituted: boolean; cells: RawLayerCellDTO[] }>` |
| Purpose | Single-source raw layer for the layer-toggle feature. |
| Inputs | `sourceKey` (already validated as 3-value by the schema), `period` |
| Output | Shape per API spec 1.2 |
| Throws | None (invalid `sourceKey` never reaches this function — caught at the schema layer) |
| Side effects | Read-only `SignalValue.findMany` filtered by `dataSourceId` (resolved from `sourceKey`) and period |
| Edge cases | Same period-fallback and empty-result handling as `getCompositeScoreLayer`. |

Test file: `tests/services/map.service.test.ts`

#### parseNaturalLanguageQuery

| Field | Detail |
|---|---|
| Signature | `parseNaturalLanguageQuery(query: string): Promise<{ parsedFilters: ParsedFilters \| null; cells: { cellId: string; compositeScore: number }[] }>` |
| Purpose | Parses free text into map filters via the LLM, then applies them. |
| Inputs | `query` (already length-validated by the schema) |
| Output | Shape per API spec 1.2 |
| Throws | `ApiError(400, "Search is temporarily unavailable")` — only when the AI service itself is unreachable/times out, distinct from "couldn't confidently parse" |
| Side effects | External LLM call (via `aiClient.ts`), then a read-only query against `GridCell`/`CompositeScoreSnapshot` using the parsed filters |
| Edge cases | LLM returns low-confidence or malformed filter output → treat as "couldn't parse," return `parsedFilters: null, cells: []` with the explanatory message (this is a **200**, not an error — see API spec 1.2). This distinction (low-confidence parse vs. service unreachable) must be enforced here, in the service, not left to the controller to guess from a generic catch block. |

Test file: `tests/services/map.service.test.ts` — mock `aiClient` for both a confident parse and a low-confidence/null parse.

---

### src/controllers/map.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| getCells | `mapService.getCompositeScoreLayer(req.query.period)` | 200, `SuccessResponse(200, "OK", result)` |
| getCellDetail | `mapService.getCellDetail(req.params.cellId, req.query.period)` | 200, `SuccessResponse(200, "OK", result)` |
| getRawLayer | `mapService.getRawSignalLayer(req.params.sourceKey, req.query.period)` | 200, `SuccessResponse(200, "OK", result)` |
| searchMap | `mapService.parseNaturalLanguageQuery(req.body.query)` | 200; message is `"OK"` normally, or the service's explanatory "couldn't understand" message when `parsedFilters` is `null` — controller branches on this one field, same pattern as the e-commerce template's capped-quantity message |

---

### src/routes/map.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | / | `validate(getCellsQuerySchema)` | getCells |
| GET | /:cellId | `validate(getCellDetailParamsSchema)` | getCellDetail |
| GET | /layers/:sourceKey | `validate(getRawLayerParamsSchema)` | getRawLayer |
| POST | /search | `validate(searchMapBodySchema)` | searchMap |

No `authMiddleware` anywhere in this router — fully public. Mounted at `/map`.

---

**Next:** proceed to → [8-3. Backend: Case Studies & Validation]
