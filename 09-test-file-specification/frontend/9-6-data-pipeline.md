## Project: Shadow Economy Map — Frontend Test Documentation: Data Pipeline Management
**Links back to:** [8-6. Frontend Function-Level Spec: Data Pipeline Management], [9-1. Frontend Test Documentation: Shared/Config], [9-5. Frontend Test Documentation: Admin Access]

Per the Frontend Testing Guide: hooks mock `@/lib/axios`; pages/components mock this feature's own hooks (`@/hooks/useAdminPipeline`). Every hook/page in this module is Admin-authenticated (per API spec 5.1) — none of these tests need to simulate the auth guard itself (that's 9-1's `ProtectedRoute` coverage); they assume an authenticated context, consistent with how 9-1's guard tests set store state directly rather than logging in through the UI.

**Flagged open item, carried over from Doc 4/8-2's own pending items:** whether creating a new `ScoreWeightConfig` (`useCreateWeightConfig`) makes it immediately active, or requires a separate activation step, is not yet settled in the backend spec (Doc 4 §4.2: "weights must sum to 1.0" and the exact formula are both still open). The test cases below for `useCreateWeightConfig`'s cache-invalidation behavior are written against the safer, currently-documented assumption (invalidates `WEIGHT_CONFIGS` only) — **do not silently assume it also invalidates map-layer queries** until that backend detail is confirmed; if it's confirmed, add that case explicitly rather than guessing it into an existing test.

---

### 9.1 Test File Map

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| `src/hooks/useAdminPipeline.ts` | `tests/hooks/useAdminPipeline.test.ts` | Unit (`renderHook`, mocked `@/lib/axios`) | ☐ |
| `src/components/admin-pipeline/SourceHealthCard.tsx` | `tests/components/admin-pipeline/SourceHealthCard.test.tsx` | Component (RTL, mocked `useTriggerRefresh`) | ☐ |
| `src/components/admin-pipeline/PipelineRunsTable.tsx` | `tests/components/admin-pipeline/PipelineRunsTable.test.tsx` | Component (RTL, presentational) | ☐ |
| `src/components/admin-pipeline/RecomputeButton.tsx` | `tests/components/admin-pipeline/RecomputeButton.test.tsx` | Component (RTL, mocked `useTriggerRecompute`) | ☐ |
| `src/components/admin-pipeline/WeightConfigForm.tsx` | `tests/components/admin-pipeline/WeightConfigForm.test.tsx` | Component (RTL, mocked `useCreateWeightConfig`) | ☐ |
| `src/components/admin-pipeline/WeightConfigHistory.tsx` | `tests/components/admin-pipeline/WeightConfigHistory.test.tsx` | Component (RTL, presentational) | ☐ |
| `src/pages/admin/AdminDashboardPage.tsx` | `tests/pages/admin/AdminDashboardPage.test.tsx` | Component (RTL, mocked `usePipelineSources`) | ☐ |
| `src/pages/admin/PipelineManagementPage.tsx` | `tests/pages/admin/PipelineManagementPage.test.tsx` | Component (RTL, mocked pipeline hooks) | ☐ |
| `src/pages/admin/WeightConfigPage.tsx` | `tests/pages/admin/WeightConfigPage.test.tsx` | Component (RTL, mocked weight-config hooks) | ☐ |

---

### 9.2 Test Case Detail — useAdminPipeline.test.ts

#### usePipelineSources

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Fetches all data sources with health status | mocked `api.get` resolves `{sources: [4 entries]}` | `renderHook(() => usePipelineSources())` | `api.get` called with `('/admin/pipeline/sources')`; `result.current.data.sources` has 4 entries |
| Every documented healthStatus value round-trips correctly | mocked `api.get` resolves sources with each of `healthy`/`stale`/`failed`/`never_run` | render, await | `result.current.data.sources` preserves each status string unchanged — this hook does no client-side reinterpretation of health status |

#### useTriggerRefresh

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Calls the correct per-source refresh endpoint | mocked `api.post` resolves | `renderHook(() => useTriggerRefresh())`, `.mutate('VIIRS')` | `api.post` called with `('/admin/pipeline/sources/VIIRS/refresh')` |
| On success, invalidates the sources and runs queries | spy on `invalidateQueries` | `.mutate('GDELT')`, await success | `invalidateQueries` called for both `[QUERY_KEYS.PIPELINE_SOURCES]` and `[QUERY_KEYS.PIPELINE_RUNS]` — a refresh changes both the source's health/last-run timestamp and appends a new run-log row |
| A failed pull still invalidates sources (to reflect the new "failed" status) | mocked `api.post` resolves successfully at the HTTP layer (per UC-12, a failed *pull* is still a successful *API call* that records a FAILED `PipelineRun`) | `.mutate('RWI')`, await success | same invalidation as the success case — the distinction between "pull succeeded" and "pull failed" lives in the *data* returned by the next `usePipelineSources`/`usePipelineRuns` fetch, not in whether this mutation itself resolves or rejects |
| Genuine request failure (network/500) surfaces as isError | mocked `api.post` rejects | `.mutate('VIIRS')` | `result.current.isError === true`; no invalidation occurs |

#### usePipelineRuns

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Fetches the run log | mocked `api.get` resolves a list of runs | `renderHook(() => usePipelineRuns())` | `api.get` called with `('/admin/pipeline/runs')` |
| Preserves failed-run errorMessage | mocked `api.get` resolves a run with `status: 'FAILED', errorMessage: 'source API unavailable'` | render, await | that run's `errorMessage` is present unchanged in `result.current.data` |

#### useTriggerRecompute

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Calls the recompute endpoint | mocked `api.post` resolves | `renderHook(() => useTriggerRecompute())`, `.mutate()` | `api.post` called with `('/admin/pipeline/recompute')` |
| On success, does not itself invalidate any public map query | spy on `invalidateQueries` | `.mutate()`, await success | no assertion of a `MAP_CELLS`/`CELL_DETAIL` invalidation here — per the flagged-open-item note above, this hook's own documented scope is the admin-side recompute trigger only; if cross-feature invalidation is later confirmed as intended behavior, that's a new, explicit test case, not an assumption baked into this one |

#### useWeightConfigs / useCreateWeightConfig

| Case | Setup | Action | Expected result |
|---|---|---|---|
| useWeightConfigs fetches the config history | mocked `api.get` resolves a list of configs, each with `weights: [{sourceKey, weight}]` | `renderHook(() => useWeightConfigs())` | `api.get` called with `('/admin/pipeline/weight-configs')` |
| Exactly one config is ever marked active in a realistic response | mocked `api.get` resolves configs where only one has `isActive: true` | render, await | `result.current.data` passes through unmodified — this hook does no active/inactive recomputation of its own; it displays whatever the backend states |
| useCreateWeightConfig calls the correct endpoint with 3 source weights | mocked `api.post` resolves a new config | `renderHook(() => useCreateWeightConfig())`, `.mutate({weights: [{sourceKey:'VIIRS', weight: 0.4}, {sourceKey:'GHSL', weight:0.3}, {sourceKey:'RWI', weight:0.3}]})` | `api.post` called with `('/admin/pipeline/weight-configs', {weights: [...]})` — exactly 3 entries, never a 4th GDELT entry, consistent with Doc 4's "GDELT never given a SourceWeight row" |
| On success, invalidates the weight-configs query | spy on `invalidateQueries` | `.mutate(...)`, await success | `invalidateQueries` called for `[QUERY_KEYS.WEIGHT_CONFIGS]` |
| Validation-shaped failure propagates without side effects | mocked `api.post` rejects with a 400 (e.g. weights not summing to an expected total, if/when that validation is confirmed backend-side) | `.mutate(...)` | `result.current.isError === true`; no invalidation occurs |

---

### 9.2 Test Case Detail — SourceHealthCard.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders name, healthStatus, and lastSuccessfulRunAt | props: a `DataSourceStatus` with `healthStatus: 'healthy'` | render | name, a "healthy"-associated visual treatment, and a formatted last-run timestamp are shown |
| Distinct treatment per healthStatus | render 4 cards, one per status value | render | `healthy`/`stale`/`failed`/`never_run` each render with a visually distinguishable treatment from the others |
| never_run renders no timestamp, not a broken date | props: `{lastSuccessfulRunAt: null, healthStatus: 'never_run'}` | render | a "never run"-equivalent fallback shown instead of attempting to format `null` |
| Refresh control triggers the mutation for this source's key | `vi.mock('@/hooks/useAdminPipeline')`; mocked `useTriggerRefresh().mutate` | `userEvent.click` the refresh control | `mutate` called with this card's specific `key` (e.g. `'VIIRS'`), not a hardcoded/wrong source |
| Refresh control shows a pending state while in flight | mocked `useTriggerRefresh` state `isPending: true` | render | refresh control disabled/shows a spinner |

---

### 9.2 Test Case Detail — PipelineRunsTable.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders one row per run, with timestamp/source/status | props: `runs: [3 PipelineRun objects]` | render | 3 rows, each showing that run's source, status, and timestamp |
| Renders errorMessage only for FAILED runs | props: a mix of `SUCCESS` and `FAILED` runs, the failed one carrying `errorMessage` | render | only the failed row displays the error text; successful rows do not render an empty error cell with stray formatting |
| Empty run log renders an empty-state row/message, not a crash | props: `runs: []` | render | a "no runs yet"-equivalent message shown, no table-row error |

---

### 9.2 Test Case Detail — RecomputeButton.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Clicking triggers the recompute mutation | `vi.mock('@/hooks/useAdminPipeline')`; mocked `useTriggerRecompute().mutate` | `userEvent.click` | `mutate` called |
| Disabled and shows a loading state while pending | mocked hook state `isPending: true` | render | button disabled, loading indicator visible |
| Shows a success confirmation after a completed run | mocked hook state `isSuccess: true` | render | a success message/indicator is shown, distinct from the idle state |

---

### 9.2 Test Case Detail — WeightConfigForm.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders exactly 3 weight fields (VIIRS, GHSL, RWI) | mocked `useCreateWeightConfig` idle state | render | 3 numeric inputs present, labeled per source; no 4th (GDELT) field exists anywhere in the form |
| Client-side validation blocks a non-numeric weight | mocked hook `mutate` | fill a weight field with non-numeric text, submit | `mutate` NOT called; a field-level validation error shown |
| Valid submit calls mutate with all 3 weights | mocked hook `mutate` | fill all 3 fields validly, submit | `mutate` called with an array/object containing all 3 `{sourceKey, weight}` entries |
| Submission loading state disables the form | mocked hook state `isPending: true` | render | submit control disabled |
| Server-side validation error (e.g. weights rejected) sets a root-level message | mocked hook error state | trigger error handling | a root-level error message shown — this project has not yet confirmed the exact validation rule (see the module-level flagged-open-item note), so this case asserts only that *some* server-rejected submission surfaces an error, not a specific numeric constraint message |

---

### 9.2 Test Case Detail — WeightConfigHistory.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders one entry per past config, newest first or per given order | props: `configs: [3 ScoreWeightConfig objects]` | render | 3 entries shown, each listing its 3 source weights and `createdAt` |
| The active config is visually highlighted | props: configs where exactly one has `isActive: true` | render | that one entry carries a distinguishing "active" treatment; the others do not |
| No config marked active renders with no highlight (defensive case) | props: all configs `isActive: false` | render | component renders without throwing and without incorrectly highlighting an arbitrary entry — guards against an implicit "highlight the first one" fallback that would misrepresent state |

---

### 9.3 Test Case Detail — AdminDashboardPage.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders a SourceHealthCard per data source | `vi.mock('@/hooks/useAdminPipeline')` → mocked `usePipelineSources` success with 4 sources | render `<AdminDashboardPage />` | 4 `SourceHealthCard`s rendered |
| Loading state | mocked hook `{isPending: true}` | render | loading indicator shown, no cards yet |
| Error state | mocked hook `{isError: true}` | render | error message shown, not a blank dashboard |

---

### 9.3 Test Case Detail — PipelineManagementPage.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Composes source cards, runs table, and recompute button | mocked `usePipelineSources`, `usePipelineRuns` success states | render `<PipelineManagementPage />` | `SourceHealthCard`s, `PipelineRunsTable`, and `RecomputeButton` all present |
| Loading states are independent per data source | mocked `usePipelineSources` success but `usePipelineRuns` still `isPending: true` | render | source cards render fully while the runs table shows its own loading state — one hook's loading state doesn't block the other section from rendering |

---

### 9.3 Test Case Detail — WeightConfigPage.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Composes the create form and history | mocked `useWeightConfigs` success, `useCreateWeightConfig` idle | render `<WeightConfigPage />` | `WeightConfigForm` and `WeightConfigHistory` both present |
| A new config appears in history after a successful create | mocked `useCreateWeightConfig` simulating its own `onSuccess` (which the hook-level test already covers invalidating `WEIGHT_CONFIGS`); mocked `useWeightConfigs` re-resolving with the new entry after invalidation | submit the form, resolve | the new config now appears in the rendered `WeightConfigHistory` — this is the one page-level test worth exercising the create → refetch loop end-to-end at the mock level, since it's the behavior an admin actually cares about |

---

### 9.4 Coverage Honesty Check (per PR Steward, at review time)

- [ ] `useTriggerRefresh`'s invalidation test checks both `PIPELINE_SOURCES` and `PIPELINE_RUNS` were invalidated — not just one of the two, since a refresh visibly changes both a source's health card and the run log
- [ ] The "failed pull is still a successful mutation" case is tested as its own distinct case, not folded into the plain success case — a reviewer should be able to see at a glance that this project's contract is "the HTTP call succeeding" and "the underlying pull succeeding" are different things
- [ ] `WeightConfigForm`'s 3-fields-only assertion explicitly queries for a 4th/GDELT field and asserts it's absent, not merely that 3 fields happen to be present
- [ ] `WeightConfigHistory`'s "no config active" defensive case is a real test, not skipped as "shouldn't happen in practice" — Doc 4 itself only says "only one config *should* be active," not that the frontend can assume the invariant always holds
- [ ] The `useCreateWeightConfig` cache-invalidation test does NOT assert anything about map-layer/public-map queries being invalidated, matching the module-level flagged-open-item note — a test that quietly encodes an unconfirmed assumption here would be worse than no test at all, since it would look like confirmed, tested behavior to a future reader
- [ ] `PipelineManagementPage`'s independent-loading-states case genuinely mocks the two hooks with different states (one pending, one resolved) rather than both in the same state, which wouldn't actually prove independence

---

### 9.5 Out of Scope for Automated Testing (and why)

- **Real Google Earth Engine / GDELT pull behavior and timing** — this frontend suite mocks `@/lib/axios` entirely; whether a real refresh actually completes, how long it takes, and what a real failure looks like are backend/infrastructure concerns (see the backend test doc's equivalent out-of-scope note).
- **The actual weight-config validation rule** (sum-to-1.0 or otherwise) — flagged throughout this document as an open backend decision (Doc 4 §4.2, Doc 8-2); this frontend suite intentionally does not encode a guess at the exact numeric rule until it's confirmed, per the module-level note above.
- **Real-time reflection of a recompute on the live public map** — whether/when a recompute becomes visible on `/map` without a manual page refresh is a caching/invalidation design question not yet settled for this project (see `useTriggerRecompute`'s test note); once settled, it becomes a new, explicit cross-feature test case, not an assumption retrofitted into this file.

---

**Next:** proceed to → [9-7. Frontend Test Documentation: Case Study Curation]
