## Project: Shadow Economy Map — Frontend Folder & File Structure

This is the file list to check every PR against. Base structure inherited from `template-react`. Per the frontend template's standing rule, every hook gets a mirrored test file under `tests/`.

**See `9-ui-foundation-spec.md` for the full design-token, common-component, and layout spec — this file only lists what exists where; that file defines what each foundation piece contains and why.**

> **Spacing namespace rule:** Project-specific spacing tokens use the `space-*` namespace to avoid collisions with Tailwind v4's built-in `sm`, `md`, `lg`, `xl`, etc. names.
>
> Examples:
> - `mt-space-sm`
> - `gap-space-md`
> - `p-space-lg`
> - `px-space-gutter`
> - `py-space-margin-mobile`
>
> Do **not** use project spacing utilities such as `mt-sm`, `gap-md`, `p-lg`, or `p-xl`.
>
> Standard Tailwind utilities remain unchanged. For example, `md:flex`, `sm:flex-row`, `gap-4`, `px-6`, and `max-w-md` are normal Tailwind utilities and must not be renamed.

---

### 7.1 Pages

**Public**

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/pages/LandingPage.tsx` | Page | intro + highlight stats | useLandingContent, HighlightStats |
| `src/pages/map/GrowthMapPage.tsx` | Page | the core map experience | useMapCells, useCellDetail, useRawLayer, useMapSearch, GrowthMapCanvas, TimeSlider, LayerToggle, MapSearchBar, CellDetailPanel, OnboardingOverlay |
| `src/pages/CaseStudiesListPage.tsx` | Page | browsable list of validated case studies | useCaseStudies, CaseStudyListRow, EmptyState |
| `src/pages/CaseStudyDetailPage.tsx` | Page | full evidence detail for one case study | useCaseStudyDetail, CaseStudyDetailPanel |
| `src/pages/MethodologyPage.tsx` | Page | scoring explanation, sources, limitations | useMethodologyContent, DataSourceCard, LimitationsList |

**Admin Auth**

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/pages/admin/AdminLoginPage.tsx` | Page | admin login form | useAdminLogin |

**Admin**

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/pages/admin/AdminDashboardPage.tsx` | Page | pipeline overview at a glance | usePipelineSources, SourceHealthCard |
| `src/pages/admin/PipelineManagementPage.tsx` | Page | trigger refreshes, view run history | usePipelineRuns, useTriggerRefresh, useTriggerRecompute, PipelineRunsTable, RecomputeButton |
| `src/pages/admin/WeightConfigPage.tsx` | Page | manage composite score weighting | useWeightConfigs, useCreateWeightConfig, WeightConfigForm, WeightConfigHistory |
| `src/pages/admin/CaseStudyCurationPage.tsx` | Page | CRUD + AI-assisted discovery for case studies | useAdminCaseStudies, useCreateCaseStudy, useUpdateCaseStudy, useDeleteCaseStudy, useDiscoverCandidates, CaseStudyTable, CaseStudyForm, DiscoveryPanel |

---

### 7.2 Layouts & Route Guards

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/components/layouts/PublicLayout.tsx` | Layout | public chrome, no auth awareness — new addition beyond the base template (see frontend conventions 0.1) | TopNavBar, Footer |
| `src/components/layouts/DashboardLayout.tsx` | Layout | modified from template placeholder: composes AdminSidebar + mobile header, renders `<Outlet />` | AdminSidebar |
| `src/routes/index.tsx` | Routes | modified to add PublicLayout (ungated) alongside the template's existing AuthLayout/PublicRoute and DashboardLayout/ProtectedRoute pairs | all layouts/guards |

`AuthLayout`, `ProtectedRoute`, `PublicRoute` are used unmodified from the base template.

**Common nav/chrome components** (see `9-ui-foundation-spec.md` §9.4 for full spec):

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/components/common/TopNavBar.tsx` | Component | public nav, active-link styling via `useLocation()` | ROUTES |
| `src/components/common/Footer.tsx` | Component | static public footer | — |
| `src/components/common/AdminSidebar.tsx` | Component | 220px admin sidebar nav + active-link styling + Logout action. No "Settings" link — present in Stitch design but not backed by any spec'd route | useAdminLogout, ROUTES |
| `src/components/common/StatusBadge.tsx` | Component | consolidated success/warning/error pill, replaces per-page hardcoded badge colors | — |

---

### 7.3 Feature Components

**Growth Map**

| File | Type | Purpose |
|---|---|---|
| `src/components/map/GrowthMapCanvas.tsx` | Component | renders the map, shades cells, handles click |
| `src/components/map/TimeSlider.tsx` | Component | controls selected period |
| `src/components/map/CellDetailPanel.tsx` | Component | click-to-inspect panel content |
| `src/components/map/LayerToggle.tsx` | Component | switches raw layer (VIIRS/GHSL/RWI only) |
| `src/components/map/MapSearchBar.tsx` | Component | natural-language search input |

**Case Studies**

| File | Type | Purpose |
|---|---|---|
| `src/components/case-studies/CaseStudyMarker.tsx` | Component | map pin — rendered inside GrowthMapCanvas, owned here |
| `src/components/case-studies/CaseStudyDetailPanel.tsx` | Component | evidence + dates + before/after |
| `src/components/case-studies/BeforeAfterSlider.tsx` | Component | drag-reveal image comparison |
| `src/components/case-studies/CaseStudyListRow.tsx` | Component | row in the list page |

**Site Content**

| File | Type | Purpose |
|---|---|---|
| `src/components/content/HighlightStats.tsx` | Component | landing page stat row |
| `src/components/content/DataSourceCard.tsx` | Component | one card per data source on methodology page |
| `src/components/content/LimitationsList.tsx` | Component | plainly-rendered limitations list |
| `src/components/content/OnboardingOverlay.tsx` | Component | static, dismissible map legend overlay |

**Admin — Data Pipeline**

| File | Type | Purpose |
|---|---|---|
| `src/components/admin-pipeline/SourceHealthCard.tsx` | Component | per-source status + refresh trigger |
| `src/components/admin-pipeline/PipelineRunsTable.tsx` | Component | paginated run history |
| `src/components/admin-pipeline/RecomputeButton.tsx` | Component | triggers score recomputation |
| `src/components/admin-pipeline/WeightConfigForm.tsx` | Component | create new weight config |
| `src/components/admin-pipeline/WeightConfigHistory.tsx` | Component | past configs, active one highlighted |

**Admin — Case Study Curation**

| File | Type | Purpose |
|---|---|---|
| `src/components/admin-case-studies/CaseStudyTable.tsx` | Component | published/draft list, edit/delete |
| `src/components/admin-case-studies/CaseStudyForm.tsx` | Component | create/edit form, surfaces date-order warning inline |
| `src/components/admin-case-studies/DiscoveryPanel.tsx` | Component | AI-assisted candidate search, pre-fills form only — never auto-publishes |

---

### 7.4 Store

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/store/adminAuth.store.ts` | Store | Zustand, persisted `admin-auth-storage` — token + admin identity | — |

---

### 7.5 Hooks

| File | Type | Purpose | Depends on |
|---|---|---|---|
| `src/hooks/useMap.ts` | Hook | useMapCells, useCellDetail, useRawLayer, useMapSearch | src/lib/axios.ts |
| `tests/hooks/useMap.test.ts` | Test | mirrors useMap.ts | — |
| `src/hooks/useCaseStudies.ts` | Hook | useCaseStudies, useCaseStudyDetail | src/lib/axios.ts |
| `tests/hooks/useCaseStudies.test.ts` | Test | mirrors useCaseStudies.ts | — |
| `src/hooks/useContent.ts` | Hook | useLandingContent, useMethodologyContent | src/lib/axios.ts |
| `tests/hooks/useContent.test.ts` | Test | mirrors useContent.ts | — |
| `src/hooks/useAdminAuth.ts` | Hook | useAdminLogin, useAdminLogout, useAdminMe | src/lib/axios.ts, adminAuth.store.ts |
| `tests/hooks/useAdminAuth.test.ts` | Test | mirrors useAdminAuth.ts | — |
| `src/hooks/useAdminPipeline.ts` | Hook | usePipelineSources, useTriggerRefresh, usePipelineRuns, useTriggerRecompute, useWeightConfigs, useCreateWeightConfig | src/lib/axios.ts |
| `tests/hooks/useAdminPipeline.test.ts` | Test | mirrors useAdminPipeline.ts | — |
| `src/hooks/useAdminCaseStudies.ts` | Hook | useAdminCaseStudies, useCreateCaseStudy, useUpdateCaseStudy, useDeleteCaseStudy, useDiscoverCandidates | src/lib/axios.ts |
| `tests/hooks/useAdminCaseStudies.test.ts` | Test | mirrors useAdminCaseStudies.ts — includes discovery-never-auto-publishes case | — |

---

### 7.6 Schema/Config Changes

| File | Change |
|---|---|
| `src/types/index.ts` | Add all types from the 6 frontend feature files (GrowthCell, CellDetail, CaseStudyDetail, LandingContent, DataSourceStatus, ScoreWeightConfig, AdminCaseStudy, etc.) |
| `src/constants/index.ts` | Add QUERY_KEYS: MAP_CELLS, CELL_DETAIL, MAP_LAYER, CASE_STUDIES, CASE_STUDY_DETAIL, CONTENT_LANDING, CONTENT_METHODOLOGY, PIPELINE_SOURCES, PIPELINE_RUNS, WEIGHT_CONFIGS, ADMIN_CASE_STUDIES. Add ROUTES for all pages in 7.1. Remove the template's placeholder DASHBOARD route (replaced by our actual admin routes) |
| `src/lib/axios.ts` | Modify the 401 interceptor to only redirect when the failing request URL matches `/admin/` (see frontend conventions 0.6) — public requests never trigger a login redirect |
| `.env.example` | Confirm VITE_API_URL includes `/api/v1` — no new frontend env vars beyond the template's defaults |

---

### 7.7 File Creation Order

**Phase A — UI Foundation**  
See `9-ui-foundation-spec.md` §9.7 for full reasoning. This must complete before Phase B.

1. `src/styles/globals.css` (modify — design tokens and namespaced project spacing tokens)
2. `index.html` (modify — font links)
3. `src/lib/utils.ts` (unchanged from template)
4. `src/components/ui/`: Button, Card, Input, Label, Checkbox, Badge
5. `src/components/common/StatusBadge.tsx`, `EmptyState.tsx`

**Phase B — Shared Config**  
Routing/state must exist before nav components can build ROUTES-aware active-link logic.

6. `src/types/index.ts` (modify)
7. `src/constants/index.ts` (modify)
8. `src/store/adminAuth.store.ts`
9. `src/lib/axios.ts` (modify)

**Phase C — Chrome & Layouts**

10. `src/components/common/TopNavBar.tsx`, `Footer.tsx`, `AdminSidebar.tsx`
11. `src/components/layouts/PublicLayout.tsx`
12. `src/components/layouts/DashboardLayout.tsx` (modify)
13. `src/routes/index.tsx` (modify — depends on all layouts/guards above)

**Phase D — Features**  
Unchanged from original order, renumbered.

14. `src/hooks/useContent.ts`
15. `src/components/content/HighlightStats.tsx`
16. `src/pages/LandingPage.tsx`
17. `src/hooks/useMap.ts`
18. `src/components/map/GrowthMapCanvas.tsx`, `TimeSlider.tsx`, `LayerToggle.tsx`, `MapSearchBar.tsx`
19. `src/hooks/useCaseStudies.ts`
20. `src/components/case-studies/CaseStudyMarker.tsx`, `CaseStudyDetailPanel.tsx`, `BeforeAfterSlider.tsx`
21. `src/components/map/CellDetailPanel.tsx` (depends on useMap)
22. `src/components/content/OnboardingOverlay.tsx`
23. `src/pages/map/GrowthMapPage.tsx`
24. `src/components/case-studies/CaseStudyListRow.tsx`
25. `src/pages/CaseStudiesListPage.tsx`, `CaseStudyDetailPage.tsx`
26. `src/components/content/DataSourceCard.tsx`, `LimitationsList.tsx`
27. `src/pages/MethodologyPage.tsx`
28. `src/hooks/useAdminAuth.ts`
29. `src/pages/admin/AdminLoginPage.tsx`
30. `src/hooks/useAdminPipeline.ts`
31. `src/components/admin-pipeline/SourceHealthCard.tsx`, `PipelineRunsTable.tsx`, `RecomputeButton.tsx`
32. `src/pages/admin/AdminDashboardPage.tsx`, `PipelineManagementPage.tsx`
33. `src/components/admin-pipeline/WeightConfigForm.tsx`, `WeightConfigHistory.tsx`
34. `src/pages/admin/WeightConfigPage.tsx`
35. `src/hooks/useAdminCaseStudies.ts`
36. `src/components/admin-case-studies/CaseStudyTable.tsx`, `CaseStudyForm.tsx`, `DiscoveryPanel.tsx`
37. `src/pages/admin/CaseStudyCurationPage.tsx`
38. Matching test files for every hook, per the standing rule in §7.5