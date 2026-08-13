## Project: Shadow Economy Map — Frontend Function-Level Spec: Shared / Config
Covers every frontend file not owned by a single feature — types, constants, the full routing tree, and template files this project modifies.

---

### src/types/index.ts (modify)

```typescript
export interface DataSourceSignal {
  source: 'VIIRS' | 'GHSL' | 'RWI';
  rawValue: number;
  normalizedValue: number;
}

export interface GrowthCell {
  cellId: string;
  cellRow: number;
  cellCol: number;
  boundaryGeoJson: GeoJSON.Polygon;
  compositeScore: number;
  isComplete: boolean;
}

export interface CellDetail {
  cellId: string;
  period: string;
  areaLabel: string | null;
  compositeScore: number;
  isComplete: boolean;
  trend: 'up' | 'down' | 'flat';
  sparkline: { period: string; compositeScore: number }[];
  signals: DataSourceSignal[];
  aiSummary: string | null;
  lastUpdated: string;
}

export interface RawLayerCell {
  cellId: string;
  normalizedValue: number;
}

export interface CaseStudySummary {
  id: string;
  name: string;
  latitude: number;
  longitude: number;
  scoreRiseDate: string;
  confirmedDate: string;
  evidenceTier: 'OFFICIAL' | 'MARKET_REPORT' | 'INFRASTRUCTURE' | 'LOCAL_NEWS';
}

export interface CaseStudyDetail extends CaseStudySummary {
  evidenceDescription: string;
  evidenceUrl: string | null;
  beforeImageUrl: string | null;
  afterImageUrl: string | null;
}

export interface LandingContent {
  tagline: string;
  intro: string;
  highlightStats: { publishedCaseStudyCount: number; lastDataRefresh: string | null };
}

export interface MethodologyContent {
  scoreExplanation: string;
  dataSources: { key: string; name: string; description: string }[];
  validationApproach: string;
  limitations: string[];
}

export interface AdminIdentity {
  id: string;
  email: string;
}

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

export interface AdminCaseStudy {
  id: string;
  name: string;
  isPublished: boolean;
  evidenceTier: string | null;
  createdAt: string;
}

export interface DiscoveryCandidate {
  summary: string;
  sourceUrl: string;
  suggestedEvidenceTier: string;
  mentionedDate: string;
}
```

No `User`/`role` type exists — V1 has no public accounts, only `AdminIdentity`, unlike the e-commerce template's `User` extension.

---

### src/constants/index.ts (modify)

```typescript
export const ROUTES = {
  HOME: '/',
  MAP: '/map',
  CASE_STUDIES: '/case-studies',
  CASE_STUDY_DETAIL: '/case-studies/:caseStudyId',
  METHODOLOGY: '/methodology',
  ADMIN_LOGIN: '/admin/login',
  ADMIN_DASHBOARD: '/admin',
  ADMIN_PIPELINE: '/admin/pipeline',
  ADMIN_WEIGHT_CONFIGS: '/admin/weight-configs',
  ADMIN_CASE_STUDIES: '/admin/case-studies',
} as const;

export const QUERY_KEYS = {
  MAP_CELLS: 'map-cells',
  CELL_DETAIL: 'cell-detail',
  MAP_LAYER: 'map-layer',
  CASE_STUDIES: 'case-studies',
  CASE_STUDY_DETAIL: 'case-study-detail',
  CONTENT_LANDING: 'content-landing',
  CONTENT_METHODOLOGY: 'content-methodology',
  ADMIN_ME: 'admin-me',
  PIPELINE_SOURCES: 'pipeline-sources',
  PIPELINE_RUNS: 'pipeline-runs',
  WEIGHT_CONFIGS: 'weight-configs',
  ADMIN_CASE_STUDIES: 'admin-case-studies',
} as const;
```
Template's placeholder `DASHBOARD`/`USERS`-style entries removed entirely — replaced, not extended, since V1's route/query shape has no overlap with the template's generic example.

---

### src/routes/index.tsx (modify)

```tsx
<BrowserRouter>
  <Suspense fallback={<PageLoader />}>
    <Routes>
      {/* Public — no guard, PublicLayout */}
      <Route element={<PublicLayout />}>
        <Route path={ROUTES.HOME} element={<LandingPage />} />
        <Route path={ROUTES.MAP} element={<GrowthMapPage />} />
        <Route path={ROUTES.CASE_STUDIES} element={<CaseStudiesListPage />} />
        <Route path={ROUTES.CASE_STUDY_DETAIL} element={<CaseStudyDetailPage />} />
        <Route path={ROUTES.METHODOLOGY} element={<MethodologyPage />} />
      </Route>

      {/* Admin login only — PublicRoute + AuthLayout */}
      <Route element={<PublicRoute />}>
        <Route element={<AuthLayout />}>
          <Route path={ROUTES.ADMIN_LOGIN} element={<AdminLoginPage />} />
        </Route>
      </Route>

      {/* Admin area — ProtectedRoute + DashboardLayout */}
      <Route element={<ProtectedRoute />}>
        <Route element={<DashboardLayout />}>
          <Route path={ROUTES.ADMIN_DASHBOARD} element={<AdminDashboardPage />} />
          <Route path={ROUTES.ADMIN_PIPELINE} element={<PipelineManagementPage />} />
          <Route path={ROUTES.ADMIN_WEIGHT_CONFIGS} element={<WeightConfigPage />} />
          <Route path={ROUTES.ADMIN_CASE_STUDIES} element={<CaseStudyCurationPage />} />
        </Route>
      </Route>

      <Route path="*" element={<NotFoundPage />} />
    </Routes>
  </Suspense>
</BrowserRouter>
```
All pages remain `React.lazy()`-loaded per the template's convention. `PublicRoute`'s "redirect away if already authenticated" behavior applies only to `/admin/login` — there is no equivalent public-facing redirect anywhere else, since public routes have no logged-in state to redirect away from.

---

### src/lib/axios.ts (modify)

| Field | Detail |
|---|---|
| Change | Modify the 401-handling interceptor: only trigger the redirect-to-login behavior when `error.config.url` matches `/admin/`; redirect target is `ROUTES.ADMIN_LOGIN`, not a generic `/login` |
| Reasoning | Public endpoints never return 401 (per API conventions 0.1) — an unmatched interceptor firing on some future public 401 (should one ever occur due to a bug) must not send an anonymous visitor to an admin login page they have no reason to see |
| Token attach | Request interceptor attaches `Authorization: Bearer <token>` from `useAdminAuthStore.getState().token` only when a token is present — public calls proceed without the header, no separate unauthenticated client instance needed |

No `useAuth.ts` hook exists in this project — there's no generic "auth" concept spanning multiple user types the way the e-commerce template's did; admin auth is fully covered by `useAdminAuth.ts` (see `04-admin-access-frontend.md`), so this file is not part of the shared/config layer here.

---

### src/components/layouts/DashboardLayout.tsx (modify)

| Field | Detail |
|---|---|
| Change | Template ships this as a placeholder wrapper. Add admin-facing nav: links to `ROUTES.ADMIN_DASHBOARD`, `ROUTES.ADMIN_PIPELINE`, `ROUTES.ADMIN_WEIGHT_CONFIGS`, `ROUTES.ADMIN_CASE_STUDIES`, and a "Logout" action calling `useAdminLogout().mutate()` |
| Behavior | Purely presentational nav additions — renders `<Outlet />` for the matched child route, unchanged from the template's own structure |
| Edge cases | None — no badge counts or live data in this nav, unlike the e-commerce template's cart-badge case |

---

### .env.example (frontend) — confirm, no new variables

`VITE_API_URL` must include the backend's versioned prefix, e.g. `http://localhost:3000/api/v1`. No project-specific frontend env vars beyond the template's defaults (`VITE_API_URL`, `VITE_APP_NAME`, `VITE_APP_ENV`).

---

**Next:** proceed to → [9-2. Frontend: Growth Map & Exploration]
