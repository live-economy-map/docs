## Project: Shadow Economy Map — Frontend Function-Level Spec: Data Pipeline Management
Files covered: `src/pages/admin/AdminDashboardPage.tsx` · `src/pages/admin/PipelineManagementPage.tsx` · `src/pages/admin/WeightConfigPage.tsx` · `src/components/admin-pipeline/*` · `src/hooks/useAdminPipeline.ts`

All routes and components in this feature sit behind `ProtectedRoute` + `DashboardLayout` (per frontend conventions 0.1) — no public access anywhere in this file.

---

### src/hooks/useAdminPipeline.ts (new — full block, per frontend spec 5.3)

#### usePipelineSources

| Field | Detail |
|---|---|
| Signature | `usePipelineSources(): UseQueryResult<{ sources: DataSourceStatus[] }>` |
| Purpose | Wraps `GET /admin/pipeline/sources`. |
| Query key | `[QUERY_KEYS.PIPELINE_SOURCES]` |
| Polling | `refetchInterval: 10_000` — refresh runs (`POST .../refresh`) are asynchronous (`202`), so this hook polls rather than waiting on the mutation response to reflect the new `healthStatus`, per frontend spec 5.3 and API spec 5.2 |
| Edge cases | A source with `lastSuccessfulRunAt: null` and `healthStatus: 'never_run'` (e.g., GDELT before its first pull) renders normally, not as an error state |
| Test file | `tests/hooks/useAdminPipeline.test.ts` |

#### useTriggerRefresh

| Field | Detail |
|---|---|
| Signature | `useTriggerRefresh(): UseMutationResult<{ pipelineRunId: string; status: 'RUNNING' }, AxiosError, string>` (variable: `sourceKey`) |
| Purpose | Wraps `POST /admin/pipeline/sources/:sourceKey/refresh`. |
| Side effects | `onSuccess` invalidates `[QUERY_KEYS.PIPELINE_SOURCES]` — the subsequent poll (above) picks up the `RUNNING` → `SUCCESS`/`FAILED` transition once the async pull completes; this hook does not itself poll the run to completion |
| Edge cases | A `409 "A refresh for this source is already in progress"` is a genuine mutation error, surfaced by `SourceHealthCard` as an inline message (not a toast that could be missed), since it means the admin's click had no effect and the UI must say so explicitly |
| Test file | `tests/hooks/useAdminPipeline.test.ts` — includes the 409-already-running case |

#### usePipelineRuns

| Field | Detail |
|---|---|
| Signature | `usePipelineRuns(sourceKey?: string, page = 1): UseQueryResult<{ items: PipelineRun[]; total: number }>` |
| Purpose | Wraps `GET /admin/pipeline/runs`. |
| Query key | `[QUERY_KEYS.PIPELINE_RUNS, sourceKey, page]` |
| Edge cases | `sourceKey: undefined` returns runs across all sources — `PipelineRunsTable`'s "all sources" filter option maps to this, not to four separate calls |
| Test file | `tests/hooks/useAdminPipeline.test.ts` |

#### useTriggerRecompute

| Field | Detail |
|---|---|
| Signature | `useTriggerRecompute(): UseMutationResult<{ period: string; scoreWeightConfigId: string }, AxiosError, string>` (variable: `period`, ISO date) |
| Purpose | Wraps `POST /admin/pipeline/recompute`. |
| Side effects | No query invalidation of map-facing queries from here — this hook lives in the admin bundle and has no reference to the public `QUERY_KEYS.MAP_CELLS` cache; a visitor's already-open map tab picks up new scores on its own next fetch, not via cross-tab invalidation |
| Edge cases | A `409 "No active weight configuration — create one first"` is a genuine mutation error — `RecomputeButton` must surface this distinctly from a generic failure, since the fix is actionable (go create a weight config) rather than "try again" |
| Test file | `tests/hooks/useAdminPipeline.test.ts` — includes the 409-no-active-config case |

#### useWeightConfigs

| Field | Detail |
|---|---|
| Signature | `useWeightConfigs(): UseQueryResult<{ configs: ScoreWeightConfig[] }>` |
| Purpose | Wraps `GET /admin/pipeline/weight-configs`. |
| Query key | `[QUERY_KEYS.WEIGHT_CONFIGS]` |
| Edge cases | Empty `configs: []` (no weight config has ever been created) is a valid, renderable state — `WeightConfigHistory` shows an empty-history message, and `RecomputeButton`/`useTriggerRecompute` callers should expect the 409 case above until an admin creates the first config |
| Test file | `tests/hooks/useAdminPipeline.test.ts` |

#### useCreateWeightConfig

| Field | Detail |
|---|---|
| Signature | `useCreateWeightConfig(): UseMutationResult<ScoreWeightConfig, AxiosError, { sourceKey: string; weight: number }[]>` |
| Purpose | Wraps `POST /admin/pipeline/weight-configs`. |
| Side effects | `onSuccess` invalidates `[QUERY_KEYS.WEIGHT_CONFIGS]` — the newly created config is immediately marked active server-side (deactivating the prior one, per API spec 5.2), so the invalidated list re-fetch is sufficient; no optimistic update needed |
| Edge cases | Three distinct `400` conditions from the API (missing/duplicate source, weights-not-summing-to-1.0 *(pending confirmation per Doc 4's open item)*, `GDELT` included) are all field-specific Zod-shaped errors except the sum-to-1.0 and GDELT checks, which are business-rule `400`s with fixed messages — `WeightConfigForm` maps the fixed-message cases to a form-level banner rather than trying to attach them to one input, since they're not single-field errors |
| Test file | `tests/hooks/useAdminPipeline.test.ts` — includes the GDELT-rejected-from-weights case |

---

### src/components/admin-pipeline/SourceHealthCard.tsx (new — full block)

| Field | Detail |
|---|---|
| Props | `{ source: DataSourceStatus }` |
| Local state | None — the "Refresh" button is wired to a shared `useTriggerRefresh()` mutation instance from the parent, or its own instance keyed by `source.key`; either way, only one in-flight refresh per card |
| Behavior | Renders `name`, a `healthStatus` badge (`healthy \| stale \| failed \| never_run`, distinct color per state), `lastSuccessfulRunAt` (formatted, or "Never" if `null`), and a "Refresh" button calling `useTriggerRefresh().mutate(source.key)` |
| Edge cases | Button is disabled while a refresh for *this* source is `RUNNING` (derived from the polled `usePipelineSources` data, not local state, so a refresh triggered from another tab/session is also reflected) — prevents the 409 case from `useTriggerRefresh` from being the primary way a double-click is caught, though the mutation still handles it defensively if the disabled state is stale for a moment |
| Test file | `tests/components/admin-pipeline/SourceHealthCard.test.tsx` |

### src/components/admin-pipeline/PipelineRunsTable.tsx (new — full block)

| Field | Detail |
|---|---|
| Props | `{ sourceKey?: string }` — `undefined` shows all sources |
| Local state | `page: number` |
| Behavior | `usePipelineRuns(sourceKey, page)`; renders `dataSourceKey`, `status`, `startedAt`, `completedAt`, `recordsProcessed`, and `errorMessage` (only shown when `status === 'FAILED'`) as a paginated table |
| Edge cases | `status: 'RUNNING'` rows (no `completedAt` yet) render a spinner/pending indicator in place of a duration, not a blank cell |
| Test file | `tests/components/admin-pipeline/PipelineRunsTable.test.tsx` |

### src/components/admin-pipeline/RecomputeButton.tsx (new — light block)

| Field | Detail |
|---|---|
| Props | `{ period: string }` — the period to recompute, selected elsewhere on the page |
| Local state | None |
| Behavior | Calls `useTriggerRecompute().mutate(period)` on click; on `202` success, shows a brief "Recomputation started" confirmation (it does not poll to completion itself — the admin checks `PipelineRunsTable`/map results separately, since recompute has no dedicated run-status endpoint of its own beyond the pipeline runs log) |
| Edge cases | On the `409` "no active weight configuration" case, renders an inline message with a link/shortcut to `ROUTES.ADMIN_WEIGHT_CONFIGS` rather than a bare error string, since the fix is a specific, known next action |
| Test file | `tests/components/admin-pipeline/RecomputeButton.test.tsx` |

### src/components/admin-pipeline/WeightConfigForm.tsx (new — full block)

| Field | Detail |
|---|---|
| Props | `{ onSuccess?: () => void }` |
| Local state | React Hook Form state — one numeric field per scoreable source (`VIIRS`, `GHSL`, `RWI`); form is statically shaped to exactly these three, mirroring `SourceWeight`'s expectation of exactly 3 rows per config (per Doc 4) — the form does not dynamically add/remove source rows |
| Behavior | Client-side Zod validation mirrors the backend's `createWeightConfigSchema`: all three sources required, weight in a sane range (e.g., 0–1), and — pending the same confirmation flagged in Doc 4/API spec 5.2 — a sum-to-1.0 check, implemented so it is trivial to remove if that requirement is dropped rather than confirmed. On submit, calls `useCreateWeightConfig().mutate(weights)`. |
| Edge cases | Client-side validation is a UX convenience only — it does not replace the server's checks (per frontend spec 5.4's "mirrors backend validation, doesn't replace it"); a `400` response still surfaces normally even if client validation nominally passed (e.g., if the sum-to-1.0 rule changes server-side before the frontend is updated) |
| Test file | `tests/components/admin-pipeline/WeightConfigForm.test.tsx` |

### src/components/admin-pipeline/WeightConfigHistory.tsx (new — light block)

| Field | Detail |
|---|---|
| Props | `{ configs: ScoreWeightConfig[] }` |
| Local state | None |
| Behavior | Lists past configs from `useWeightConfigs`'s `configs` (most recent first, per API spec 5.2's ordering), each showing its `weights` breakdown and `createdAt`; the entry with `isActive: true` is visually highlighted, and only one such entry should ever be highlighted at a time |
| Edge cases | `configs: []` — renders an empty-history message, not a blocked page (an admin can still create the first config via `WeightConfigForm` on the same page) |
| Test file | `tests/components/admin-pipeline/WeightConfigHistory.test.tsx` |

---

### src/pages/admin/AdminDashboardPage.tsx (new)

| Field | Detail |
|---|---|
| Purpose | Pipeline health overview at a glance — the admin's landing page after login. |
| Route guard + layout | `ProtectedRoute`, `DashboardLayout` |
| Local state | None |
| Behavior | `usePipelineSources()`; renders one `SourceHealthCard` per source in `sources` |
| Components | SourceHealthCard |

**States:** loading (skeleton cards) · error (inline + retry) · success (polling continues per the hook's `refetchInterval`)

### src/pages/admin/PipelineManagementPage.tsx (new)

| Field | Detail |
|---|---|
| Purpose | Trigger refreshes and review run history in one place (UC-12). |
| Route guard + layout | `ProtectedRoute`, `DashboardLayout` |
| Local state | `sourceFilter: string \| undefined`, `selectedPeriod: string` (for the recompute control) |
| Behavior | Renders `PipelineRunsTable` (filtered by `sourceFilter`) and `RecomputeButton` (for `selectedPeriod`); source refresh triggers live on this page's `SourceHealthCard`-style controls or are reachable from `AdminDashboardPage` — the two pages share `usePipelineSources`/`useTriggerRefresh` rather than duplicating fetch logic |
| Components | PipelineRunsTable, RecomputeButton |

**States:** loading · error (inline + retry, independent per section — a runs-table failure does not block the recompute control) · success

### src/pages/admin/WeightConfigPage.tsx (new)

| Field | Detail |
|---|---|
| Purpose | Manage composite score weighting for experimentation/tuning (UC-13). |
| Route guard + layout | `ProtectedRoute`, `DashboardLayout` |
| Local state | None |
| Behavior | `useWeightConfigs()`; renders `WeightConfigForm` (create) and `WeightConfigHistory` (list, `configs` passed down) |
| Components | WeightConfigForm, WeightConfigHistory |

**States:** loading (skeleton history) · error (inline + retry) · empty (`configs: []` → history section shows empty message, form still usable) · success

---

**Next:** proceed to → [9-7. Frontend: Case Study Curation]
