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

**Correction:** the previous version of this doc showed the classic `<BrowserRouter>`/`<Routes>`/`<Route>` component API. The actual template (arch doc v3.1, Section 2.5 / 9.1) uses the **data router API** — `createBrowserRouter` building a route-config array, mounted via `<RouterProvider>` in `src/App.tsx`, with per-route Suspense via the template's `withSuspense()` helper rather than one top-level `<Suspense>`. The block below matches that.

```tsx
import { createBrowserRouter } from 'react-router-dom';
import { withSuspense } from './withSuspense'; // template helper, unchanged
import PublicLayout from '@/components/layouts/PublicLayout';
import AuthLayout from '@/components/layouts/AuthLayout';
import DashboardLayout from '@/components/layouts/DashboardLayout';
import ProtectedRoute from './ProtectedRoute';
import PublicRoute from './PublicRoute';
import { ROUTES } from '@/constants';
// ...lazy-loaded page imports, same as template convention

export const router = createBrowserRouter([
  {
    element: <PublicLayout />,
    children: [
      { path: ROUTES.HOME, element: withSuspense(LandingPage) },
      { path: ROUTES.MAP, element: withSuspense(GrowthMapPage) },
      { path: ROUTES.CASE_STUDIES, element: withSuspense(CaseStudiesListPage) },
      { path: ROUTES.CASE_STUDY_DETAIL, element: withSuspense(CaseStudyDetailPage) },
      { path: ROUTES.METHODOLOGY, element: withSuspense(MethodologyPage) },
    ],
  },
  {
    element: <PublicRoute />,
    children: [
      {
        element: <AuthLayout />,
        children: [{ path: ROUTES.ADMIN_LOGIN, element: withSuspense(AdminLoginPage) }],
      },
    ],
  },
  {
    element: <ProtectedRoute />,
    children: [
      {
        element: <DashboardLayout />,
        children: [
          { path: ROUTES.ADMIN_DASHBOARD, element: withSuspense(AdminDashboardPage) },
          { path: ROUTES.ADMIN_PIPELINE, element: withSuspense(PipelineManagementPage) },
          { path: ROUTES.ADMIN_WEIGHT_CONFIGS, element: withSuspense(WeightConfigPage) },
          { path: ROUTES.ADMIN_CASE_STUDIES, element: withSuspense(CaseStudyCurationPage) },
        ],
      },
    ],
  },
  { path: '*', element: withSuspense(NotFoundPage) },
]);

export default router;
```
All pages remain lazy-loaded, now via `withSuspense()` per-route rather than one shared `<Suspense>` boundary. `RouterProvider router={router}` is mounted in `src/App.tsx`, per the template's existing wiring — this project does not change that part. `PublicRoute`'s "redirect away if already authenticated" behavior applies only to `/admin/login` — there is no equivalent public-facing redirect anywhere else, since public routes have no logged-in state to redirect away from. `Outlet`/`Navigate` inside each layout/guard component are unchanged from the template.

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
| Change | Template ships this as a placeholder wrapper. This project composes it from `components/common/AdminSidebar.tsx` (nav to `ROUTES.ADMIN_DASHBOARD`, `ROUTES.ADMIN_PIPELINE`, `ROUTES.ADMIN_WEIGHT_CONFIGS`, `ROUTES.ADMIN_CASE_STUDIES`, and Logout calling `useAdminLogout().mutate()`) plus an inline mobile header — full component spec in `9-ui-foundation-spec.md` §9.4–9.5 |
| Behavior | Layout itself stays thin: renders `AdminSidebar`, the mobile header, and `<Outlet />` for the matched child route. The active-link logic and Logout wiring live in `AdminSidebar`, not here |
| Edge cases | None — no badge counts or live data in this nav, unlike the e-commerce template's cart-badge case |

---

### src/styles/globals.css, index.html, components/ui/*, components/common/* (new/modify)

Design tokens, font loading, and the shared component set (`TopNavBar`, `Footer`, `AdminSidebar`, `StatusBadge`, `EmptyState`, themed `ui/` primitives) are specified in full in **`9-ui-foundation-spec.md`** — not duplicated here. That doc also defines the required build order for this layer, which precedes everything in this file's routing/layout sections.

---

### .env.example (frontend) — confirm, no new variables

`VITE_API_URL` must include the backend's versioned prefix, e.g. `http://localhost:3000/api/v1`. No project-specific frontend env vars beyond the template's defaults (`VITE_API_URL`, `VITE_APP_NAME`, `VITE_APP_ENV`).

---

**Next:** proceed to → [9-2. Frontend: Growth Map & Exploration]
