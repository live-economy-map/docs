## Project: Shadow Economy Map — Backend Test Documentation: Growth Map & Exploration
**Links back to:** [8-2. Function-Level Spec: Growth Map & Exploration]

Per the standing rule in `7a-backend-file-structure.md`: test-first, test file mirrors `src/` exactly under `tests/`, never co-located. Vitest — `describe`/`it`/`expect`, mocks via `vi.fn()`/`vi.mock()`, `beforeEach(() => vi.clearAllMocks())`.

---

### 9.1 Test File Map

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| src/schemas/map.schema.ts | tests/schemas/map.schema.test.ts | Unit (schema parsing) | ☐ |
| src/services/map.service.ts | tests/services/map.service.test.ts | Unit (mocked Prisma via `vi.mock('@/lib/prisma')`, mocked `aiClient.ts`) | ☐ |
| src/controllers/map.controller.ts | tests/controllers/map.controller.test.ts | Unit (mocked map.service via `vi.mock`) | ☐ |
| src/routes/map.routes.ts | tests/routes/map.routes.test.ts | Integration (supertest, mocked service layer) | ☐ |

---

### 9.2 Test Case Detail — map.schema.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| getCellsQuerySchema accepts a valid ISO date period | — | parse `{ query: { period: "2026-06-01" } }` | passes; `period` unchanged |
| getCellsQuerySchema accepts an omitted period | — | parse `{ query: {} }` | passes; `period` undefined — resolution of "most recent" happens in the service, not the schema |
| getCellsQuerySchema rejects a malformed period | — | parse `{ query: { period: "not-a-date" } }` | validation fails |
| getCellDetailParamsSchema requires cellId as uuid | — | parse `{ params: { cellId: "not-a-uuid" }, query: {} }` | validation fails |
| getRawLayerParamsSchema accepts VIIRS/GHSL/RWI | — | parse `{ params: { sourceKey: "GHSL" }, query: {} }` | passes |
| getRawLayerParamsSchema rejects GDELT | — | parse `{ params: { sourceKey: "GDELT" }, query: {} }` | validation fails — confirms the 3-value enum, not the full 4-value `DataSourceKey`, per API spec 1.2's explicit exclusion |
| getRawLayerParamsSchema rejects an unknown sourceKey | — | parse `{ params: { sourceKey: "LANDSAT" }, query: {} }` | validation fails |
| searchMapBodySchema rejects a query under 3 chars | — | parse `{ body: { query: "hi" } }` | validation fails |
| searchMapBodySchema rejects a query over 200 chars | — | parse `{ body: { query: "a".repeat(201) } }` | validation fails |
| searchMapBodySchema accepts a valid query | — | parse `{ body: { query: "areas near Bole with rising construction" } }` | passes |

---

### 9.2 Test Case Detail — map.service.test.ts

#### getCompositeScoreLayer

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Requested period has data | mock `prisma.compositeScoreSnapshot.findMany` to resolve cells for the exact requested period, filtered to the active `ScoreWeightConfig` | call `getCompositeScoreLayer("2026-06-01")` | resolves `{ period: "2026-06-01", periodSubstituted: false, cells: [...] }` |
| Requested period has no data — falls back | mock `findMany` for the requested period to resolve `[]`, then mock a lookup that finds the nearest earlier period with rows | call `getCompositeScoreLayer("2026-07-01")` | resolves the nearest earlier period's cells with `periodSubstituted: true`, and the returned `period` reflects the substituted period, not the originally requested one |
| No data exists for any period | mock every relevant query to resolve `[]`/`null` | call `getCompositeScoreLayer("2026-07-01")` | resolves `{ cells: [], periodSubstituted: false }`, does not throw — per UC-01's alternate flow, this is a rendered empty state, not a 404 |
| No period argument — defaults to most recent | mock a "most recent period with data" lookup | call `getCompositeScoreLayer()` | resolves the latest available period's cells, confirming `undefined` input resolves to "most recent," not an error |
| Query scoped to active weight config only | mock two `CompositeScoreSnapshot` rows for the same cell/period under two different `scoreWeightConfigId`s, only one `isActive: true` | call `getCompositeScoreLayer(period)` | assert the Prisma mock was called with a filter tying the query to the currently active `ScoreWeightConfig` — not just that the return value happens to look right, since a query bug here could silently mix scores from two different formulas |

#### getCellDetail

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Cell exists with full data | mock `GridCell` lookup, 3 `SignalValue` rows, current + prior `CompositeScoreSnapshot`, last 6 periods for sparkline, mock `generateCellSummary` to resolve a string | call `getCellDetail(cellId, period)` | resolves full `CellDetailDTO` — composite score, all 3 raw signals, `trend`, non-empty `sparkline`, `areaLabel`, `aiSummary` populated |
| Cell doesn't exist | mock `GridCell` lookup → `null` | call `getCellDetail(badCellId, period)` | throws `ApiError(404, "No data available for this cell")` |
| Cell exists but has zero snapshots across all periods | mock `GridCell` found, but `CompositeScoreSnapshot.findMany` (all periods) → `[]` | call `getCellDetail(cellId, period)` | throws the same `ApiError(404, "No data available for this cell")` |
| No prior-period snapshot — trend defaults to flat | mock current-period snapshot present, prior-period lookup → `null` | call `getCellDetail(cellId, period)` | resolves with `trend: 'flat'`, does not throw or guess a direction |
| generateCellSummary fails — rest of payload still returned | mock `generateCellSummary` (or the underlying `aiClient` call it wraps) to resolve `null` | call `getCellDetail(cellId, period)` | resolves with `aiSummary: null`, all numeric fields still fully populated — confirms the AI failure never blocks the rest of the response, per Doc 2's AI-reliability NFR |
| generateCellSummary called with the right inputs | mock `generateCellSummary`; spy on its call args | call `getCellDetail(cellId, period)` | assert `generateCellSummary` was called with the resolved signal values and composite score, not empty/placeholder data |

#### generateCellSummary

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Successful summary | mock `aiClient` (however its wrapper is named, per Doc 9-1's assumed interface) to resolve a summary string | call `generateCellSummary(signals, compositeScore)` | resolves the string unchanged |
| aiClient throws — swallowed, not propagated | mock `aiClient` call to reject | call `generateCellSummary(signals, compositeScore)` | resolves `null`; does **not** throw — this is the one function in the backend explicitly designed to swallow its own errors (per Doc 8-2), and a test that lets a thrown error escape here should fail |
| aiClient times out — same swallowed behavior | mock `aiClient` call to reject with a timeout-shaped error | call `generateCellSummary(signals, compositeScore)` | resolves `null`, same as any other `aiClient` failure — this function does not distinguish timeout from other failure modes, unlike `parseNaturalLanguageQuery` below |
| Never propagates a raw aiClient error object | mock `aiClient` to reject with a non-`Error` value (e.g., a raw SDK error shape) | call `generateCellSummary(signals, compositeScore)` | resolves `null` — confirms the catch is unconditional, not narrowed to only `Error` instances |

#### getRawSignalLayer

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Valid source, requested period has data | mock `SignalValue.findMany` filtered by resolved `dataSourceId` and period | call `getRawSignalLayer('VIIRS', period)` | resolves `{ period, periodSubstituted: false, cells: [...] }` |
| No data for requested period — falls back | mock `findMany` for the period → `[]`, mock nearest-prior-period lookup | call `getRawSignalLayer('GHSL', period)` | resolves nearest available period's cells, `periodSubstituted: true` — same fallback contract as `getCompositeScoreLayer` |
| No data for any period, for this layer only | mock every relevant query → `[]` | call `getRawSignalLayer('RWI', period)` | resolves `{ cells: [], periodSubstituted: false }`, not an error — and does not affect any other layer's data (per UC-04's alternate flow, this is scoped to the one layer) |

#### parseNaturalLanguageQuery

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Confident parse | mock `aiClient`'s structured-output call to resolve valid filters; mock the follow-up `GridCell`/`CompositeScoreSnapshot` query to resolve matching cells | call `parseNaturalLanguageQuery("areas near Bole with rising construction")` | resolves `{ parsedFilters: {...}, cells: [...] }` — a populated, non-null `parsedFilters` |
| Low-confidence / malformed parse — not an error | mock `aiClient`'s structured-output call to resolve `null` (per Doc 9-1's aiClient contract) | call `parseNaturalLanguageQuery(query)` | resolves `{ parsedFilters: null, cells: [] }` — a normal `200`-shaped success, **not** a thrown error; this is the case that must not be conflated with the AI-unreachable case below |
| AI service unreachable/times out — genuine error | mock `aiClient`'s structured-output call to reject (connection/timeout error, not a malformed-response resolve) | call `parseNaturalLanguageQuery(query)` | throws `ApiError(400, "Search is temporarily unavailable")` — this distinction (low-confidence parse vs. service down) must be enforced in the service, not left for the controller to infer from a generic catch, per Doc 8-2 |
| Empty cells for a confidently-parsed filter | mock a confident parse whose filters happen to match nothing | call `parseNaturalLanguageQuery(query)` | resolves `{ parsedFilters: {...}, cells: [] }` — a valid, non-error "your filters matched nothing" result, distinct in shape from the null-`parsedFilters` case above |

---

### 9.3 Test Case Detail — map.controller.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| getCells delegates correctly | `vi.mock` map.service; `getCompositeScoreLayer` resolves `{period, periodSubstituted, cells}` | call controller with `req.query.period` | `getCompositeScoreLayer` called with `req.query.period`; responds 200 via `SuccessResponse(200, "OK", result)` |
| getCellDetail delegates correctly | mock `getCellDetail` resolves detail object | call controller with `req.params.cellId`, `req.query.period` | `getCellDetail` called with both args in order; responds 200 with `SuccessResponse(200, "OK", result)` |
| getCellDetail propagates 404 | mock `getCellDetail` to throw `ApiError(404, "No data available for this cell")` | call controller | error passed through to error-handling middleware via `asyncHandler`, not swallowed |
| getRawLayer delegates correctly | mock `getRawSignalLayer` resolves layer object | call controller with `req.params.sourceKey`, `req.query.period` | `getRawSignalLayer` called with both args; responds 200 |
| searchMap responds with "OK" when parsedFilters is non-null | mock `parseNaturalLanguageQuery` resolves `{ parsedFilters: {...}, cells: [...] }` | call controller with `req.body.query` | responds 200 with message `"OK"` |
| searchMap responds with the explanatory message when parsedFilters is null | mock `parseNaturalLanguageQuery` resolves `{ parsedFilters: null, cells: [] }` | call controller | responds 200 (not an error status) with the service's explanatory "couldn't understand" message — branches on the `parsedFilters` field, same pattern as the e-commerce template's capped-quantity message |
| searchMap propagates the 400 unavailable error | mock `parseNaturalLanguageQuery` to throw `ApiError(400, "Search is temporarily unavailable")` | call controller | error passed through unchanged, not rewritten into the low-confidence message |

---

### 9.3 Test Case Detail — map.routes.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| GET / has no auth requirement | mock controller layer, no Authorization header | request `GET /map` | 200, not 401 — confirms fully public |
| GET / validates period format | mock controller layer | request `GET /map?period=not-a-date` | rejected by `validate(getCellsQuerySchema)`, controller mock never called |
| GET /:cellId requires a uuid cellId | mock controller layer | request `GET /map/not-a-uuid` | rejected by `validate(getCellDetailParamsSchema)`, controller mock never called |
| GET /layers/:sourceKey rejects GDELT at the route layer | mock controller layer | request `GET /map/layers/GDELT` | rejected by `validate(getRawLayerParamsSchema)` — never reaches the controller, confirming the exclusion is enforced before the service even has a chance to run |
| GET /layers/:sourceKey accepts VIIRS/GHSL/RWI | mock controller layer | request `GET /map/layers/VIIRS` | 200, controller mock invoked |
| POST /search enforces min/max query length | mock controller layer | request `POST /map/search` with `{ query: "" }` | rejected by `validate(searchMapBodySchema)`, controller mock never called |
| POST /search has no auth requirement | mock controller layer, no Authorization header | request `POST /map/search` with a valid body | 200, not 401 |
| No route in this router requires auth | mock controller layer, no Authorization header | request each of GET /, GET /:cellId, GET /layers/:sourceKey, POST /search | none return 401 — confirms this entire router is public, per Doc 8-2 |

---

### 9.4 Coverage Honesty Check (per PR Steward, at review time)

- [ ] The period-fallback logic (`periodSubstituted: true`) is tested against a case where the requested period genuinely has no data — not just asserted on a mock that always returns the substituted flag regardless of input
- [ ] `parseNaturalLanguageQuery`'s two distinct failure paths — low-confidence parse (`parsedFilters: null`, 200) vs. AI service down (`400` thrown) — are tested as genuinely different code paths, not collapsed into one generic try/catch test
- [ ] `generateCellSummary`'s error-swallowing is tested by actually making the mocked `aiClient` call reject, not by asserting the function's return type allows `null`
- [ ] The GDELT-rejected-as-raw-layer case is tested at both the schema layer (400) and confirmed absent from the service layer (i.e., `getRawSignalLayer` is never even given the chance to run with `'GDELT'`) — per the file map note in Doc 8-2 ("`GDELT` is a schema-level 400 here, not a runtime check in the service")
- [ ] `getCellDetail`'s "cell exists but zero snapshots" 404 and "cell doesn't exist" 404 are both tested with the identical `ApiError` assertion, not assumed to be the same from one case alone
- [ ] No test hardcodes a `cells` array that isn't actually derived from the mocked Prisma response

---

### 9.5 Out of Scope for Automated Testing (and why)

- **Real LLM output quality/tone** (e.g., whether generated summaries actually avoid forecasting language) — unit tests mock `aiClient` entirely; verifying the system prompt genuinely keeps the model from saying "will grow"/"is expected to" needs manual/spot-check review against real model output, per Doc 2's AI-reliability NFR and Doc 8-2's note that this is enforced via prompt design, not re-validated in code.
- **Actual map rendering performance** (the 3-second initial load / 1.5-second time-slider NFRs from Doc 2) — these are frontend/browser performance concerns, not backend unit-test concerns; tracked separately as manual timing checks.
- **Real Prisma query correctness against a live database** (join behavior, period-fallback query efficiency at full grid scale) — unit tests verify branching logic against mocked Prisma only; a real-database integration pass is tracked separately.

---

**Next:** proceed to → [9-3. Backend Test Documentation: Case Studies & Validation]
