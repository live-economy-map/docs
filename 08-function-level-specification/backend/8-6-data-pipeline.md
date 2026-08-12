## Project: Shadow Economy Map — Backend Function-Level Spec: Data Pipeline Management
Files covered: `src/schemas/adminPipeline.schema.ts` · `src/services/adminPipeline.service.ts` · `src/controllers/adminPipeline.controller.ts` · `src/routes/adminPipeline.routes.ts` · `src/utils/dataSourceClients/*`

---

### src/schemas/adminPipeline.schema.ts (new)

| Schema | Shape |
|---|---|
| refreshParamsSchema | `z.object({ params: z.object({ sourceKey: z.nativeEnum(DataSourceKey) }) })` (all 4 values valid here — unlike the map layer schema, `GDELT` is a legitimate refresh target even though it's excluded from scoring) |
| recomputeBodySchema | `z.object({ body: z.object({ period: z.string().date() }) })` |
| createWeightConfigSchema | `z.object({ body: z.object({ weights: z.array(z.object({ sourceKey: z.enum(['VIIRS','GHSL','RWI']), weight: z.number().min(0).max(1) })).length(3).refine(w => new Set(w.map(x => x.sourceKey)).size === 3, "Must include VIIRS, GHSL, and RWI exactly once each") }) })` |

`GET /admin/pipeline/sources`, `GET /admin/pipeline/runs` (query-optional `sourceKey`/pagination — validated inline, not worth a named schema given its simplicity), and `GET /admin/pipeline/weight-configs` need minimal or no schema, consistent with the e-commerce template's own judgment call on when a named schema earns its keep.

---

### src/services/adminPipeline.service.ts (new)

#### getSourcesWithHealth

| Field | Detail |
|---|---|
| Signature | `getSourcesWithHealth(): Promise<DataSourceStatusDTO[]>` |
| Purpose | All 4 sources with computed health status. |
| Inputs | None |
| Output | Shape per API spec 5.2 |
| Throws | None |
| Side effects | Read-only `DataSource.findMany()` joined to each source's most recent `PipelineRun` |
| Edge cases | A source with zero `PipelineRun` rows ever → `healthStatus: 'never_run'`, `lastSuccessfulRunAt: null`. A source whose most recent run `SUCCESS`ed long ago but hasn't been refreshed recently → `healthStatus: 'stale'`; the staleness threshold (e.g., 45 days, matched to VIIRS's monthly cadence) is a config constant, not hardcoded inline — flag as an open value to set during implementation. |

Test file: `tests/services/adminPipeline.service.test.ts`

#### startPipelineRun

| Field | Detail |
|---|---|
| Signature | `startPipelineRun(sourceKey: DataSourceKey, triggeredById: string): Promise<{ pipelineRunId: string; status: 'RUNNING' }>` |
| Purpose | Creates a `RUNNING` `PipelineRun` row, then asynchronously invokes the matching client (`viirs.client.ts` etc.) without awaiting its completion in the request/response cycle. |
| Inputs | `sourceKey`, `triggeredById` (from `req.user.id`) |
| Output | The new run's id, immediately |
| Throws | `ApiError(409, "A refresh for this source is already in progress")` — an existing `RUNNING` row for the same `dataSourceId` |
| Side effects | Creates `PipelineRun`; kicks off the actual pull (fire-and-forget from this function's perspective — the pull's own completion handler updates the `PipelineRun` row to `SUCCESS`/`FAILED` independently, writes `SignalValue` rows on success, and never throws back into this function's caller) |
| Edge cases | The async pull itself failing (network error, malformed response) is handled entirely inside the pull's own completion handler, not here — this function's job ends once the `RUNNING` row exists and the async work is dispatched. Malformed/incomplete pulled data → mark the run `FAILED` with a descriptive `errorMessage`, do not partially write `SignalValue` rows (all-or-nothing per period/source). |

Test file: `tests/services/adminPipeline.service.test.ts` — the 409-conflict case is the main synchronous behavior to test directly; the async pull completion logic is tested via the individual `dataSourceClients/*` files instead.

#### getPipelineRuns

| Field | Detail |
|---|---|
| Signature | `getPipelineRuns(sourceKey?: DataSourceKey, page?: number, limit?: number): Promise<{ items: PipelineRunDTO[]; page: number; limit: number; total: number }>` |
| Purpose | Observability log, most recent first. |
| Inputs | Optional `sourceKey` filter, pagination |
| Output | Shape per API spec 5.2 |
| Throws | None |
| Side effects | Read-only, ordered by `startedAt desc` |
| Edge cases | Standard empty-result-not-an-error pattern. |

Test file: `tests/services/adminPipeline.service.test.ts`

#### recomputeCompositeScores

| Field | Detail |
|---|---|
| Signature | `recomputeCompositeScores(period: string, triggeredById: string): Promise<{ period: string; scoreWeightConfigId: string }>` |
| Purpose | Recalculates `CompositeScoreSnapshot` for every `GridCell` for the given period, using the active `ScoreWeightConfig`. |
| Inputs | `period` |
| Output | Confirmation shape, immediately (async, per API spec 5.2's `202`) |
| Throws | `ApiError(409, "No active weight configuration — create one first")` — no `ScoreWeightConfig` row has `isActive: true` |
| Side effects | For each `GridCell`: reads the 3 `SignalValue` rows (VIIRS/GHSL/RWI) for the period, computes `Σ(weight × normalizedValue)` per the active config's `SourceWeight` rows, upserts `CompositeScoreSnapshot` on `(gridCellId, period, scoreWeightConfigId)` |
| Edge cases | A cell missing one or more of the 3 source signals for the period → still compute a score from whatever's available, but set `isComplete: false` on that cell's snapshot (per Doc 4) — never skip the cell entirely and never silently treat a missing signal as `0`, since that would understate the score rather than honestly flagging incompleteness. |

Test file: `tests/services/adminPipeline.service.test.ts` — explicitly covers the missing-signal → `isComplete: false` path, and the no-active-config 409.

#### getWeightConfigs / createAndActivateWeightConfig

| Field | Detail |
|---|---|
| Signature | `getWeightConfigs(): Promise<ScoreWeightConfigDTO[]>` · `createAndActivateWeightConfig(weights: {sourceKey: string; weight: number}[], createdById: string): Promise<ScoreWeightConfigDTO>` |
| Purpose | List/create weight configurations. |
| Inputs | (create) validated 3-source weight array, `createdById` |
| Output | Config shape(s) per API spec 5.2 |
| Throws | (create) `ApiError(400, "Weights must sum to 1.0")` — **pending confirmation**, see Doc 4's open item; may be removed or changed if a different formula is decided |
| Side effects | (create) Transaction: set any existing `isActive: true` config to `false`, insert the new `ScoreWeightConfig` + 3 `SourceWeight` rows, set it `isActive: true` — must be atomic (a single Prisma `$transaction`), since a partial write would leave two configs simultaneously active or none active |
| Edge cases | Sum-to-1.0 validation, if kept, should allow a small floating-point tolerance (e.g., `Math.abs(sum - 1) < 0.001`), not exact equality. |

Test file: `tests/services/adminPipeline.service.test.ts` — covers the atomic deactivate-then-activate transaction explicitly (e.g., simulate a mid-transaction failure and confirm no partial state).

---

### src/utils/dataSourceClients/*.ts (new)

Not given per-function tables here (external API wrapper detail is implementation-specific to each provider's SDK/REST shape) — but each client shares the same contract:

```typescript
interface DataSourceClient {
  pull(boundingBox: BoundingBox, period: string): Promise<{ cellId: string; rawValue: number }[]>;
}
```

`viirs.client.ts`/`ghsl.client.ts` wrap Google Earth Engine; `rwi.client.ts` reads the static Meta RWI dataset (no live API call — confirm during implementation whether this is a one-time import into a lookup table or a per-request file read); `gdelt.client.ts` wraps GDELT's REST API. Each has its own test file mirroring this contract, mocking the underlying external call.

---

### src/controllers/adminPipeline.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| listSources | `adminPipelineService.getSourcesWithHealth()` | 200, `SuccessResponse(200, "OK", { sources })` |
| triggerRefresh | `adminPipelineService.startPipelineRun(req.params.sourceKey, req.user.id)` | 202, `SuccessResponse(202, "Refresh started", result)` |
| listRuns | `adminPipelineService.getPipelineRuns(req.query.sourceKey, req.query.page, req.query.limit)` | 200, `SuccessResponse(200, "OK", result)` |
| triggerRecompute | `adminPipelineService.recomputeCompositeScores(req.body.period, req.user.id)` | 202, `SuccessResponse(202, "Recomputation started", result)` |
| listWeightConfigs | `adminPipelineService.getWeightConfigs()` | 200, `SuccessResponse(200, "OK", { configs })` |
| createWeightConfig | `adminPipelineService.createAndActivateWeightConfig(req.body.weights, req.user.id)` | 201, `SuccessResponse(201, "Weight configuration created and activated", config)` |

---

### src/routes/adminPipeline.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | /sources | `authMiddleware` | listSources |
| POST | /sources/:sourceKey/refresh | `authMiddleware, validate(refreshParamsSchema)` | triggerRefresh |
| GET | /runs | `authMiddleware` | listRuns |
| POST | /recompute | `authMiddleware, validate(recomputeBodySchema)` | triggerRecompute |
| GET | /weight-configs | `authMiddleware` | listWeightConfigs |
| POST | /weight-configs | `authMiddleware, validate(createWeightConfigSchema)` | createWeightConfig |

`authMiddleware` applies to every route in this router. Mounted at `/admin/pipeline`.

---

**Next:** proceed to → [8-7. Backend: Case Study Curation]
