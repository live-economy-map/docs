## Project: Shadow Economy Map — Frontend Specification
**Feature:** Data Pipeline Management
**Conventions:** see `0-frontend-conventions.md` — `ProtectedRoute` + `DashboardLayout` throughout.
**API reference:** `05-data-pipeline-api.md`

---

### 5.1 Routes
```
/admin              → ProtectedRoute → DashboardLayout → AdminDashboardPage (pipeline overview)
/admin/pipeline      → ProtectedRoute → DashboardLayout → PipelineManagementPage (runs log)
/admin/weight-configs → ProtectedRoute → DashboardLayout → WeightConfigPage
```

### 5.2 Types
```typescript
export interface DataSourceStatus {
  key: 'VIIRS' | 'GHSL' | 'RWI' | 'GDELT';
  name: string;
  isActive: boolean;
  lastSuccessfulRunAt: string | null;
  healthStatus: 'healthy' | 'stale' | 'failed' | 'never_run';
}

export interface PipelineRun {
  id: string;
  dataSourceKey: string;
  status: 'RUNNING' | 'SUCCESS' | 'FAILED';
  startedAt: string;
  completedAt: string | null;
  recordsProcessed: number | null;
  errorMessage: string | null;
}

export interface ScoreWeightConfig {
  id: string;
  isActive: boolean;
  createdAt: string;
  weights: { sourceKey: 'VIIRS' | 'GHSL' | 'RWI'; weight: number }[];
}
```

### 5.3 Hooks (src/hooks/useAdminPipeline.ts)
```typescript
export function usePipelineSources() {
  return useQuery({
    queryKey: [QUERY_KEYS.PIPELINE_SOURCES],
    queryFn: () => api.get<{ sources: DataSourceStatus[] }>('/admin/pipeline/sources').then((r) => r.data),
    refetchInterval: 10_000, // poll — refresh runs are async, per API spec 5.2
  });
}

export function useTriggerRefresh() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (sourceKey: string) => api.post(`/admin/pipeline/sources/${sourceKey}/refresh`),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.PIPELINE_SOURCES] }),
  });
}

export function usePipelineRuns(sourceKey?: string, page = 1) {
  return useQuery({
    queryKey: [QUERY_KEYS.PIPELINE_RUNS, sourceKey, page],
    queryFn: () => api.get<{ items: PipelineRun[]; total: number }>('/admin/pipeline/runs', {
      params: { sourceKey, page },
    }).then((r) => r.data),
  });
}

export function useTriggerRecompute() {
  return useMutation({
    mutationFn: (period: string) => api.post('/admin/pipeline/recompute', { period }),
  });
}

export function useWeightConfigs() {
  return useQuery({
    queryKey: [QUERY_KEYS.WEIGHT_CONFIGS],
    queryFn: () => api.get<{ configs: ScoreWeightConfig[] }>('/admin/pipeline/weight-configs').then((r) => r.data),
  });
}

export function useCreateWeightConfig() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (weights: { sourceKey: string; weight: number }[]) =>
      api.post('/admin/pipeline/weight-configs', { weights }),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.WEIGHT_CONFIGS] }),
  });
}
```

### 5.4 Components (src/components/admin-pipeline/)
| Component | Responsibility |
|---|---|
| `SourceHealthCard.tsx` | One per source; shows `healthStatus` badge + "Refresh" button wired to `useTriggerRefresh` |
| `PipelineRunsTable.tsx` | Paginated table from `usePipelineRuns` |
| `RecomputeButton.tsx` | Triggers `useTriggerRecompute` for a selected period |
| `WeightConfigForm.tsx` | React Hook Form + Zod; client-side validates the 3 required sources (VIIRS/GHSL/RWI) and, if confirmed per Doc 4's open item, that weights sum to 1.0 — mirrors backend validation, doesn't replace it |
| `WeightConfigHistory.tsx` | Lists past configs from `useWeightConfigs`, highlighting the active one |

---

**Next:** proceed to → [06. Case Study Curation Frontend]
