## Project: Shadow Economy Map — Backend Test Documentation: Data Pipeline Management
**Links back to:** [8-6. Function-Level Spec: Data Pipeline Management]

Per the standing rule in `7a-backend-file-structure.md`: test-first, test file mirrors `src/` exactly under `tests/`, never co-located. Vitest — `describe`/`it`/`expect`, mocks via `vi.fn()`/`vi.mock()`, `beforeEach(() => vi.clearAllMocks())`.

---

### 9.1 Test File Map

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| src/schemas/adminPipeline.schema.ts | tests/schemas/adminPipeline.schema.test.ts | Unit (schema parsing) | ☐ |
| src/services/adminPipeline.service.ts | tests/services/adminPipeline.service.test.ts | Unit (mocked Prisma via `vi.mock('@/lib/prisma')`, mocked `dataSourceClients/*`) | ☐ |
| src/controllers/adminPipeline.controller.ts | tests/controllers/adminPipeline.controller.test.ts | Unit (mocked adminPipeline.service via `vi.mock`) | ☐ |
| src/routes/adminPipeline.routes.ts | tests/routes/adminPipeline.routes.test.ts | Integration (supertest, mocked service layer + mocked `authMiddleware`) | ☐ |
| src/utils/dataSourceClients/viirs.client.ts | tests/utils/dataSourceClients/viirs.client.test.ts | Unit (mocked Earth Engine SDK) | ☐ |
| src/utils/dataSourceClients/ghsl.client.ts | tests/utils/dataSourceClients/ghsl.client.test.ts | Unit (mocked Earth Engine SDK) | ☐ |
| src/utils/dataSourceClients/rwi.client.ts | tests/utils/dataSourceClients/rwi.client.test.ts | Unit (mocked static dataset read) | ☐ |
| src/utils/dataSourceClients/gdelt.client.ts | tests/utils/dataSourceClients/gdelt.client.test.ts | Unit (mocked GDELT REST call) | ☐ |

---

### 9.2 Test Case Detail — adminPipeline.schema.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| refreshParamsSchema accepts all 4 DataSourceKey values, including GDELT | — | parse `{ params: { sourceKey: "GDELT" } }` | passes — unlike the map layer schema (Doc 9-2), `GDELT` is a legitimate refresh target here even though it's excluded from scoring; this is the specific asymmetry Doc 8-6 flags explicitly |
| refreshParamsSchema rejects an invalid sourceKey | — | parse `{ params: { sourceKey: "LANDSAT" } }` | validation fails |
| recomputeBodySchema requires a valid ISO date period | — | parse `{ body: { period: "not-a-date" } }` | validation fails |
| recomputeBodySchema accepts a valid period | — | parse `{ body: { period: "2026-06-01" } }` | passes |
| createWeightConfigSchema requires exactly 3 weights | — | parse `{ body: { weights: [{ sourceKey: "VIIRS", weight: 0.5 }, { sourceKey: "GHSL", weight: 0.5 }] } }` (only 2 entries) | validation fails — `.length(3)` |
| createWeightConfigSchema rejects a duplicate source | — | parse `{ body: { weights: [{sourceKey:"VIIRS",weight:0.4},{sourceKey:"VIIRS",weight:0.3},{sourceKey:"RWI",weight:0.3}] } }` | validation fails — the `.refine()` uniqueness check catches this even though the array length is correct |
| createWeightConfigSchema rejects GDELT as a weighted source | — | parse `{ body: { weights: [{sourceKey:"VIIRS",weight:0.34},{sourceKey:"GHSL",weight:0.33},{sourceKey:"GDELT",weight:0.33}] } }` | validation fails — `GDELT` isn't a member of the 3-value `sourceKey` enum in this schema, distinct from the "must include exactly VIIRS/GHSL/RWI" refine check above; both reasons should be tested since either alone would catch this input, and only testing one leaves the other regression-prone |
| createWeightConfigSchema rejects a weight outside 0–1 | — | parse `{ body: { weights: [{sourceKey:"VIIRS",weight:1.5},{sourceKey:"GHSL",weight:0.3},{sourceKey:"RWI",weight:0.3}] } }` | validation fails |
| createWeightConfigSchema accepts a valid 3-source set | — | parse `{ body: { weights: [{sourceKey:"VIIRS",weight:0.4},{sourceKey:"GHSL",weight:0.35},{sourceKey:"RWI",weight:0.25}] } }` | passes — note: the schema itself does **not** enforce sum-to-1.0 (that's a service-layer business-rule check per Doc 8-6/API spec 5.2, pending confirmation); this case intentionally uses weights that do sum to 1.0 so it isn't accidentally coupled to that unresolved question |

---

### 9.2 Test Case Detail — adminPipeline.service.test.ts

#### getSourcesWithHealth

| Case | Setup | Action | Expected result |
|---|---|---|---|
| All 4 sources, mixed health | mock `prisma.dataSource.findMany` joined to each source's most recent `PipelineRun`; vary `status`/`completedAt` per source | call `getSourcesWithHealth()` | resolves 4 entries, each with the correct `healthStatus` derived from its own most recent run |
| Source with zero PipelineRun rows ever | mock a source with no associated runs | call `getSourcesWithHealth()` | that entry has `healthStatus: 'never_run'`, `lastSuccessfulRunAt: null` |
| Source succeeded long ago, not refreshed recently | mock a source whose most recent run `SUCCESS`ed past the staleness threshold | call `getSourcesWithHealth()` | that entry has `healthStatus: 'stale'` — assert against the config constant for the threshold, not a hardcoded number duplicated in the test, so the test and implementation can't silently drift apart |
| Source's most recent run failed | mock a source whose most recent run is `FAILED` | call `getSourcesWithHealth()` | that entry has `healthStatus: 'failed'`, even if an earlier run had succeeded — confirms "most recent," not "any successful ever" |
| Source's most recent run is still RUNNING | mock a source whose most recent run has `status: 'RUNNING'`, no `completedAt` | call `getSourcesWithHealth()` | `healthStatus` reflects the prior completed run's outcome (or `never_run`/`stale` as appropriate), not a `RUNNING`-specific health value that doesn't exist in the documented enum (`healthy \| stale \| failed \| never_run`) — assert the service maps this case to one of the four defined values, not a fifth ad hoc state |

#### startPipelineRun

| Case | Setup | Action | Expected result |
|---|---|---|---|
| No existing RUNNING run for this source | mock the "existing RUNNING run" lookup → `null`; mock `PipelineRun.create` | call `startPipelineRun('VIIRS', adminId)` | resolves `{ pipelineRunId, status: 'RUNNING' }`; assert `PipelineRun.create` was called with `dataSourceId` resolved for `'VIIRS'` and `triggeredById: adminId` |
| Existing RUNNING run for the same source | mock the "existing RUNNING run" lookup to resolve a row | call `startPipelineRun('VIIRS', adminId)` | throws `ApiError(409, "A refresh for this source is already in progress")`; assert `PipelineRun.create` was **not** called — a new run row must not be created on top of a conflict |
| Existing RUNNING run for a *different* source doesn't block this one | mock a RUNNING row for `'GHSL'` only | call `startPipelineRun('VIIRS', adminId)` | resolves normally — the 409 check is scoped per `dataSourceId`, not global across all sources |
| Dispatches the pull without awaiting it in the response cycle | mock the matching data source client's `pull()` to be a long-pending, never-resolving promise during the test | call `startPipelineRun('VIIRS', adminId)` | the function still resolves (does not hang waiting on `pull()`) — confirms the fire-and-forget behavior Doc 8-6 specifies; the test should fail (timeout) if this function ever awaits the pull directly |
| Async pull failure is not this function's concern | — | (documented boundary, not a case on `startPipelineRun` itself) | Per Doc 8-6: the pull's own completion handler updates the `PipelineRun` row independently — see the separate "pull completion" cases under the individual `dataSourceClients/*` test files below, not here |

#### getPipelineRuns

| Case | Setup | Action | Expected result |
|---|---|---|---|
| No filter — all sources | mock `findMany` ordered by `startedAt desc`; mock `count` | call `getPipelineRuns()` | resolves `{ items, page: 1, limit: 20, total }`, ordered most-recent-first |
| Filtered by sourceKey | mock `findMany` scoped to one `dataSourceId` | call `getPipelineRuns('GHSL')` | assert `findMany` was called with a filter resolving `'GHSL'`'s `dataSourceId`, not just that the returned items happen to all be GHSL |
| No runs for the given filter | mock `findMany` → `[]`; `count` → `0` | call `getPipelineRuns('GDELT')` | resolves `{ items: [], total: 0 }`, not an error |

#### recomputeCompositeScores

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Active weight config exists, full data for every cell | mock an active `ScoreWeightConfig` with 3 `SourceWeight` rows; mock every `GridCell` having all 3 `SignalValue`s for the period; mock the upsert | call `recomputeCompositeScores(period, adminId)` | resolves `{ period, scoreWeightConfigId }`; assert every cell's `CompositeScoreSnapshot` upsert used `compositeScore = Σ(weight × normalizedValue)` computed from the mocked weights/values, not a hardcoded/passthrough number |
| No active weight config | mock the active-config lookup → `null` | call `recomputeCompositeScores(period, adminId)` | throws `ApiError(409, "No active weight configuration — create one first")`; assert no `CompositeScoreSnapshot` writes were attempted |
| A cell is missing one of the 3 source signals for the period | mock one `GridCell` with only VIIRS + GHSL `SignalValue` rows for the period, RWI missing | call `recomputeCompositeScores(period, adminId)` | that cell's upserted snapshot has `isComplete: false`; assert the cell is **still written** (not skipped) and its score is computed from whatever's available — a missing signal must never be silently treated as `0` in the weighted sum, per Doc 8-6's explicit warning |
| A cell is missing all 3 source signals for the period | mock a `GridCell` with zero `SignalValue` rows for the period | call `recomputeCompositeScores(period, adminId)` | that cell's snapshot is still written with `isComplete: false` — same "never skip the cell entirely" rule, taken to the extreme case |
| Upsert uses the correct unique key | spy on the upsert call args | call `recomputeCompositeScores(period, adminId)` | assert each upsert targets `(gridCellId, period, scoreWeightConfigId)`, matching Doc 4's unique composite — confirms recomputing the same period under the same active config overwrites that config's snapshot rather than creating a duplicate row or silently colliding with a different config's history |

#### getWeightConfigs / createAndActivateWeightConfig

| Case | Setup | Action | Expected result |
|---|---|---|---|
| getWeightConfigs lists most-recent-first | mock `findMany` ordered by `createdAt desc`, each with 3 `SourceWeight` rows | call `getWeightConfigs()` | resolves configs in that order, each including its `weights` array |
| createAndActivateWeightConfig — atomic deactivate-then-activate | mock `$transaction` to run its callback; mock the prior active config's update to `isActive: false` and the new config's insert with `isActive: true` | call `createAndActivateWeightConfig(weights, adminId)` | assert both writes happened inside a single `$transaction` call, not as two independent sequential calls — a sequence of two separate calls could leave the system with zero or two active configs if the process crashes between them |
| createAndActivateWeightConfig — simulated mid-transaction failure | mock `$transaction`'s callback to reject partway through (e.g., the insert succeeds but the deactivate-prior step throws) | call `createAndActivateWeightConfig(weights, adminId)` | the whole operation rejects and, per Prisma's transaction semantics, no partial state is left — assert the test's mock setup reflects a rollback (i.e., don't assert "the new config exists" after a simulated failure) |
| Sum-to-1.0 rejected — *pending confirmation, per Doc 4/8-6's open item* | mock weights that sum to e.g. 0.9 | call `createAndActivateWeightConfig(weights, adminId)` | **Only implement this case once the sum-to-1.0 requirement is confirmed.** If confirmed: throws `ApiError(400, "Weights must sum to 1.0")`. If the formula changes to not require summing to 1.0: this case should be removed rather than left in a permanently-skipped state, so a stale test doesn't imply a decision was made when it wasn't |
| Sum-to-1.0 floating-point tolerance — *same pending status* | mock weights summing to `0.9999996` (float rounding artifact of three numbers that "should" sum to 1.0) | call `createAndActivateWeightConfig(weights, adminId)` | if the sum-to-1.0 rule is confirmed: does **not** throw — per Doc 8-6's explicit tolerance note (`Math.abs(sum - 1) < 0.001`), an exact-equality check would be a bug even under the confirmed rule |

---

### 9.2 Test Case Detail — dataSourceClients/*.test.ts

Each client shares one contract (Doc 8-6): `pull(boundingBox, period): Promise<{ cellId: string; rawValue: number }[]>`. The following table applies once per client file, substituting the specific external dependency being mocked.

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Successful pull | mock the underlying SDK/REST call (Earth Engine for VIIRS/GHSL, static dataset read for RWI, REST fetch for GDELT) to resolve valid data | call `pull(boundingBox, period)` | resolves an array of `{ cellId, rawValue }`, correctly mapped from the raw provider response shape into this contract |
| External call fails/times out | mock the underlying call to reject | call `pull(boundingBox, period)` | rejects — the client itself does not swallow the error; per Doc 8-6, it's the caller (the pull's completion handler in `startPipelineRun`'s dispatched work, not the client) that marks the `PipelineRun` `FAILED` and writes `errorMessage`, so the client must propagate, not absorb, the failure |
| Malformed/incomplete provider response | mock the underlying call to resolve an unexpected shape (e.g., missing expected fields) | call `pull(boundingBox, period)` | either throws a descriptive error, or resolves a result the caller can identify as incomplete — whichever behavior is chosen, it must be distinguishable from a clean empty result (`[]`), so the completion handler can correctly flag the run as needing attention (per Doc 3 UC-12's alternate flow) rather than silently accepting bad data |
| Empty bounding box / no data in range | mock the underlying call to resolve a genuinely empty result for the given box/period | call `pull(boundingBox, period)` | resolves `[]` — a clean empty result, not an error; distinct from the malformed-response case above |
| rwi.client.ts specifically — static dataset, no live API call | mock whatever local read mechanism is chosen (file read vs. pre-imported lookup table, per Doc 8-6's open confirm-during-implementation note) | call `pull(boundingBox, period)` | resolves data without any network mock being invoked — assert no HTTP/SDK call was made, confirming this client's documented "no live API call" nature is actually upheld in code, not just in the docs |

---

### 9.3 Test Case Detail — adminPipeline.controller.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| listSources delegates correctly | `vi.mock` adminPipeline.service; `getSourcesWithHealth` resolves an array | call controller | responds 200 with `SuccessResponse(200, "OK", { sources })` |
| triggerRefresh delegates correctly and returns 202 | mock `startPipelineRun` resolves `{pipelineRunId, status}` | call controller with `req.params.sourceKey`, `req.user.id` | `startPipelineRun` called with both; responds **202** (not 200) with `SuccessResponse(202, "Refresh started", result)` |
| triggerRefresh propagates the 409 conflict | mock `startPipelineRun` to throw `ApiError(409, "A refresh for this source is already in progress")` | call controller | error passed through unchanged via `asyncHandler` |
| listRuns delegates correctly | mock `getPipelineRuns` resolves `{items, page, limit, total}` | call controller with `req.query` | `getPipelineRuns` called with `sourceKey`/`page`/`limit` from query; responds 200 |
| triggerRecompute delegates correctly and returns 202 | mock `recomputeCompositeScores` resolves `{period, scoreWeightConfigId}` | call controller with `req.body.period`, `req.user.id` | responds **202** with `SuccessResponse(202, "Recomputation started", result)` |
| triggerRecompute propagates the 409 no-active-config error | mock `recomputeCompositeScores` to throw `ApiError(409, "No active weight configuration — create one first")` | call controller | error passed through unchanged |
| listWeightConfigs delegates correctly | mock `getWeightConfigs` resolves an array | call controller | responds 200 with `SuccessResponse(200, "OK", { configs })` |
| createWeightConfig delegates correctly and returns 201 | mock `createAndActivateWeightConfig` resolves a config | call controller with `req.body.weights`, `req.user.id` | responds **201** with `SuccessResponse(201, "Weight configuration created and activated", config)` |

---

### 9.3 Test Case Detail — adminPipeline.routes.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Every route in this router requires auth | mock controller layer, no Authorization header | request each of GET /sources, POST /sources/:sourceKey/refresh, GET /runs, POST /recompute, GET /weight-configs, POST /weight-configs | all six return 401, controller mock never invoked — confirms router-level auth wiring, per Doc 8-6 ("`authMiddleware` applies to every route in this router") |
| POST /sources/:sourceKey/refresh validates sourceKey | valid auth, mock controller layer | request with `sourceKey: "LANDSAT"` | rejected by `validate(refreshParamsSchema)`, controller mock never called |
| POST /recompute validates period | valid auth, mock controller layer | request with `{ period: "not-a-date" }` | rejected by `validate(recomputeBodySchema)`, controller mock never called |
| POST /weight-configs validates weights shape | valid auth, mock controller layer | request with only 2 weight entries | rejected by `validate(createWeightConfigSchema)`, controller mock never called |
| GET /sources, GET /runs, GET /weight-configs have no body validation, only auth | valid auth, mock controller layer | request each with no query params | request reaches the controller mock — these three don't have a dedicated schema (per Doc 8-6's note on minimal/no schema for simple GETs) |

---

### 9.4 Coverage Honesty Check (per PR Steward, at review time)

- [ ] The 409-already-running case in `startPipelineRun` is scoped per-source, tested with a case proving a different source's RUNNING run does *not* block this one — not just the same-source conflict alone
- [ ] `recomputeCompositeScores`'s missing-signal case asserts `isComplete: false` **and** that the cell is still written — a test that only checks one of these two would miss a regression where incomplete cells are silently dropped
- [ ] The composite score calculation itself is asserted against a computed expected value derived from the mocked weights/signals (`Σ(weight × normalizedValue)`), not just "a number came back" — this is the core scoring logic and deserves an exact-value assertion, not a shape-only one
- [ ] The atomic-transaction test for `createAndActivateWeightConfig` actually simulates a failure mid-transaction and confirms no partial state, per Doc 8-6's explicit instruction — not just a happy-path transaction call
- [ ] The fire-and-forget behavior of `startPipelineRun` is tested by actually using a non-resolving mock for the dispatched pull, not inferred from the function's return type
- [ ] Every `dataSourceClients/*` test confirms the client **propagates** (not swallows) a failed external call — a client-level swallow here would silently break the pipeline's own failure-tracking (`PipelineRun.status = FAILED`)
- [ ] The sum-to-1.0 test cases are either genuinely implemented against a confirmed decision, or clearly absent/flagged pending — not silently guessed at

---

### 9.5 Out of Scope for Automated Testing (and why)

- **Real Google Earth Engine / GDELT API behavior** — `viirs.client.ts`, `ghsl.client.ts`, and `gdelt.client.ts` are unit-tested against mocked SDK/REST calls only; real credential auth, actual response-shape drift from these third-party providers, and rate limits need a manual or separately-tracked integration pass.
- **Real concurrent refresh-trigger races** (two admins triggering the same source's refresh at nearly the same instant) — the 409 check is unit-tested for the "already exists" case, but a genuine race condition at the database level (two `PipelineRun` creates racing the same existence check) needs load/concurrency testing against a real database, not a mocked Prisma client — same precedent as the reference project's cart-stock-race exclusion.
- **Recompute performance at full grid scale** — Doc 2's scalability NFR requires the pipeline to support the full Addis Ababa grid without redesign; unit tests verify the per-cell computation logic, not throughput/performance across the entire grid, which is a manual/perf concern.
- **Staleness-threshold tuning** (the exact number of days before a source is marked `stale`) — unit tests assert against whatever the config constant currently is, not a specific hardcoded day count; tuning that number is a product decision, not something a passing/failing test should gate.

---

**Next:** proceed to → [9-7. Backend Test Documentation: Case Study Curation]
