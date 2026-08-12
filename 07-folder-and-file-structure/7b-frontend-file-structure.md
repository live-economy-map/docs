## Project: Shadow Economy Map — Frontend Folder & File Structure
This is the file list to check every PR against. Base structure inherited from `template-react`. Per the frontend template's standing rule, every hook gets a mirrored test file under `tests/`.

---

### 7.1 Pages

**Public**
| File | Type | Purpose | Depends on |
|---|---|---|---|
| src/pages/LandingPage.tsx | Page | intro + highlight stats | useLandingContent, HighlightStats |
| src/pages/map/GrowthMapPage.tsx | Page | the core map experience | useMapCells, useCellDetail, useRawLayer, useMapSearch, GrowthMapCanvas, TimeSlider, LayerToggle, MapSearchBar, CellDetailPanel, OnboardingOverlay |
| src/pages/CaseStudiesListPage.tsx | Page | browsable list of validated case studies | useCaseStudies, CaseStudyListRow, EmptyState |
| src/pages/CaseStudyDetailPage.tsx | Page | full evidence detail for one case study | useCaseStudyDetail, CaseStudyDetailPanel |
| src/pages/MethodologyPage.tsx | Page | scoring explanation, sources, limitations | useMethodologyContent, DataSourceCard, LimitationsList |

**Admin Auth**
| File | Type | Purpose | Depends on |
|---|---|---|---|
| src/pages/admin/AdminLoginPage.tsx | Page | admin login form | useAdminLogin |

**Admin**
| File | Type | Purpose | Depends on |
|---|---|---|---|
| src/pages/admin/AdminDashboardPage.tsx | Page | pipeline overview at a glance | usePipelineSources, SourceHealthCard |
| src/pages/admin/PipelineManagementPage.tsx | Page | trigger refreshes, view run history | usePipelineRuns, useTriggerRefresh, useTriggerRecompute, PipelineRunsTable, RecomputeButton |
| src/pages/admin/WeightConfigPage.tsx | Page | manage composite score weighting | useWeightConfigs, useCreateWeightConfig, WeightConfigForm, WeightConfigHistory |
| src/pages/admin/CaseStudyCurationPage.tsx | Page | CRUD + AI-assisted discovery for case studies | useAdminCaseStudies, useCreateCaseStudy, useUpdateCaseStudy, useDeleteCaseStudy, useDiscoverCandidates, CaseStudyTable, CaseStudyForm, DiscoveryPanel |

---

### 7.2 Layouts & Route Guards

| File | Type | Purpose | Depends on |
|---|---|---|---|
| src/components/layouts/PublicLayout.tsx | Layout | public chrome, no auth awareness — new addition beyond the base template (see frontend conventions 0.1) | — |
| src/components/layouts/DashboardLayout.tsx | Layout | modified from template placeholder: admin nav (Pipeline, Weight Configs, Case Studies, Logout) | useAdminLogout |
| src/routes/index.tsx | Routes | modified to add PublicLayout (ungated) alongside the template's existing AuthLayout/PublicRoute and DashboardLayout/ProtectedRoute pairs | all layouts/guards |

`AuthLayout`, `ProtectedRoute`, `PublicRoute` are used unmodified from the base template.

---

### 7.3 Feature Components

**Growth Map**
| File | Type | Purpose |
|---|---|---|
| src/components/map/GrowthMapCanvas.tsx | Component | renders the map, shades cells, handles click |
| src/components/map/TimeSlider.tsx | Component | controls selected period |
| src/components/map/CellDetailPanel.tsx | Component | click-to-inspect panel content |
| src/components/map/LayerToggle.tsx | Component | switches raw layer (VIIRS/GHSL/RWI only) |
| src/components/map/MapSearchBar.tsx | Component | natural-language search input |

**Case Studies**
| File | Type | Purpose |
|---|---|---|
| src/components/case-studies/CaseStudyMarker.tsx | Component | map pin — rendered inside GrowthMapCanvas, owned here |
| src/components/case-studies/CaseStudyDetailPanel.tsx | Component | evidence + dates + before/after |
| src/components/case-studies/BeforeAfterSlider.tsx | Component | drag-reveal image comparison |
| src/components/case-studies/CaseStudyListRow.tsx | Component | row in the list page |

**Site Content**
| File | Type | Purpose |
|---|---|---|
| src/components/content/HighlightStats.tsx | Component | landing page stat row |
| src/components/content/DataSourceCard.tsx | Component | one card per data source on methodology page |
| src/components/content/LimitationsList.tsx | Component | plainly-rendered limitations list |
| src/components/content/OnboardingOverlay.tsx | Component | static, dismissible map legend overlay |

**Admin — Data Pipeline**
| File | Type | Purpose |
|---|---|---|
| src/components/admin-pipeline/SourceHealthCard.tsx | Component | per-source status + refresh trigger |
| src/components/admin-pipeline/PipelineRunsTable.tsx | Component | paginated run history |
| src/components/admin-pipeline/RecomputeButton.tsx | Component | triggers score recomputation |
| src/components/admin-pipeline/WeightConfigForm.tsx | Component | create new weight config |
| src/components/admin-pipeline/WeightConfigHistory.tsx | Component | past configs, active one highlighted |

**Admin — Case Study Curation**
| File | Type | Purpose |
|---|---|---|
| src/components/admin-case-studies/CaseStudyTable.tsx | Component | published/draft list, edit/delete |
| src/components/admin-case-studies/CaseStudyForm.tsx | Component | create/edit form, surfaces date-order warning inline |
| src/components/admin-case-studies/DiscoveryPanel.tsx | Component | AI-assisted candidate search, pre-fills form only — never auto-publishes |

**Common**
| File | Type | Purpose |
|---|---|---|
| src/components/common/EmptyState.tsx | Component | generic empty-state message + optional CTA |

---

### 7.4 Store

| File | Type | Purpose | Depends on |
|---|---|---|---|
| src/store/adminAuth.store.ts | Store | Zustand, persisted `admin-auth-storage` — token + admin identity | — |

---

### 7.5 Hooks

| File | Type | Purpose | Depends on |
|---|---|---|---|
| src/hooks/useMap.ts | Hook | useMapCells, useCellDetail, useRawLayer, useMapSearch | src/lib/axios.ts |
| tests/hooks/useMap.test.ts | Test | mirrors useMap.ts | — |
| src/hooks/useCaseStudies.ts | Hook | useCaseStudies, useCaseStudyDetail | src/lib/axios.ts |
| tests/hooks/useCaseStudies.test.ts | Test | mirrors useCaseStudies.ts | — |
| src/hooks/useContent.ts | Hook | useLandingContent, useMethodologyContent | src/lib/axios.ts |
| tests/hooks/useContent.test.ts | Test | mirrors useContent.ts | — |
| src/hooks/useAdminAuth.ts | Hook | useAdminLogin, useAdminLogout, useAdminMe | src/lib/axios.ts, adminAuth.store.ts |
| tests/hooks/useAdminAuth.test.ts | Test | mirrors useAdminAuth.ts | — |
| src/hooks/useAdminPipeline.ts | Hook | usePipelineSources, useTriggerRefresh, usePipelineRuns, useTriggerRecompute, useWeightConfigs, useCreateWeightConfig | src/lib/axios.ts |
| tests/hooks/useAdminPipeline.test.ts | Test | mirrors useAdminPipeline.ts | — |
| src/hooks/useAdminCaseStudies.ts | Hook | useAdminCaseStudies, useCreateCaseStudy, useUpdateCaseStudy, useDeleteCaseStudy, useDiscoverCandidates | src/lib/axios.ts |
| tests/hooks/useAdminCaseStudies.test.ts | Test | mirrors useAdminCaseStudies.ts — includes discovery-never-auto-publishes case | — |

---

### 7.6 Schema/Config Changes

| File | Change |
|---|---|
| src/types/index.ts | Add all types from the 6 frontend feature files (GrowthCell, CellDetail, CaseStudyDetail, LandingContent, DataSourceStatus, ScoreWeightConfig, AdminCaseStudy, etc.) |
| src/constants/index.ts | Add QUERY_KEYS: MAP_CELLS, CELL_DETAIL, MAP_LAYER, CASE_STUDIES, CASE_STUDY_DETAIL, CONTENT_LANDING, CONTENT_METHODOLOGY, PIPELINE_SOURCES, PIPELINE_RUNS, WEIGHT_CONFIGS, ADMIN_CASE_STUDIES. Add ROUTES for all pages in 7.1. Remove the template's placeholder DASHBOARD route (replaced by our actual admin routes) |
| src/lib/axios.ts | Modify the 401 interceptor to only redirect when the failing request URL matches `/admin/` (see frontend conventions 0.6) — public requests never trigger a login redirect |
| .env.example | Confirm VITE_API_URL includes /api/v1 — no new frontend env vars beyond the template's defaults |

---

### 7.7 File Creation Order

1. src/types/index.ts (modify)
2. src/constants/index.ts (modify)
3. src/store/adminAuth.store.ts
4. src/lib/axios.ts (modify)
5. src/components/layouts/PublicLayout.tsx
6. src/components/layouts/DashboardLayout.tsx (modify)
7. src/routes/index.tsx (modify — depends on all layouts/guards above)
8. src/hooks/useContent.ts
9. src/components/content/HighlightStats.tsx
10. src/pages/LandingPage.tsx
11. src/hooks/useMap.ts
12. src/components/map/GrowthMapCanvas.tsx, TimeSlider.tsx, LayerToggle.tsx, MapSearchBar.tsx
13. src/hooks/useCaseStudies.ts
14. src/components/case-studies/CaseStudyMarker.tsx, CaseStudyDetailPanel.tsx, BeforeAfterSlider.tsx
15. src/components/map/CellDetailPanel.tsx (depends on useMap)
16. src/components/content/OnboardingOverlay.tsx
17. src/pages/map/GrowthMapPage.tsx
18. src/components/case-studies/CaseStudyListRow.tsx
19. src/pages/CaseStudiesListPage.tsx, CaseStudyDetailPage.tsx
20. src/components/content/DataSourceCard.tsx, LimitationsList.tsx
21. src/pages/MethodologyPage.tsx
22. src/hooks/useAdminAuth.ts
23. src/pages/admin/AdminLoginPage.tsx
24. src/hooks/useAdminPipeline.ts
25. src/components/admin-pipeline/SourceHealthCard.tsx, PipelineRunsTable.tsx, RecomputeButton.tsx
26. src/pages/admin/AdminDashboardPage.tsx, PipelineManagementPage.tsx
27. src/components/admin-pipeline/WeightConfigForm.tsx, WeightConfigHistory.tsx
28. src/pages/admin/WeightConfigPage.tsx
29. src/hooks/useAdminCaseStudies.ts
30. src/components/admin-case-studies/CaseStudyTable.tsx, CaseStudyForm.tsx, DiscoveryPanel.tsx
31. src/pages/admin/CaseStudyCurationPage.tsx
32. Matching test files for every hook, per the standing rule in §7.5
