## Project: Shadow Economy Map — Backend Folder & File Structure
This is the file list to check every PR against. Base structure inherited from `template-node-express` — only files this project adds are listed.

**Deviation from the e-commerce template:** no `adminOnly.middleware.ts` — V1 has exactly one role (`Admin`), so `authMiddleware` alone is sufficient; a role check would be dead code until V2 introduces a second role.

**Standing rule:** every service file gets a mirrored test file under `tests/`, per template convention.

---

### 7.1 Backend — New Files

**Growth Map & Exploration**
| File | Type | Purpose | Depends on |
|---|---|---|---|
| src/schemas/map.schema.ts | Zod schema | getCellsQuerySchema, getCellDetailParamsSchema, getRawLayerParamsSchema, searchMapBodySchema | — |
| src/services/map.service.ts | Service | getCompositeScoreLayer, getCellDetail, generateCellSummary, getRawSignalLayer, parseNaturalLanguageQuery | prisma, aiClient.ts |
| src/controllers/map.controller.ts | Controller | getCells, getCellDetail, getRawLayer, searchMap handlers | map.service.ts |
| src/routes/map.routes.ts | Route | mounts /map/* — fully public | map.controller.ts |
| tests/services/map.service.test.ts | Test | mirrors map.service.ts — includes period-fallback and GDELT-rejected-as-layer cases | — |

**Case Studies & Validation**
| File | Type | Purpose | Depends on |
|---|---|---|---|
| src/schemas/caseStudies.schema.ts | Zod schema | listCaseStudiesQuerySchema | — |
| src/services/caseStudies.service.ts | Service | getPublishedCaseStudies, getPublishedCaseStudyById | prisma |
| src/controllers/caseStudies.controller.ts | Controller | listCaseStudies, getCaseStudyById handlers | caseStudies.service.ts |
| src/routes/caseStudies.routes.ts | Route | mounts /case-studies/* — fully public | caseStudies.controller.ts |
| tests/services/caseStudies.service.test.ts | Test | mirrors caseStudies.service.ts — includes unpublished-returns-404 case | — |

**Site Content & Onboarding**
| File | Type | Purpose | Depends on |
|---|---|---|---|
| src/services/content.service.ts | Service | getLandingStats, getMethodologyContent | prisma |
| src/controllers/content.controller.ts | Controller | getLandingContent, getMethodologyContent handlers | content.service.ts |
| src/routes/content.routes.ts | Route | mounts /content/* — fully public | content.controller.ts |
| tests/services/content.service.test.ts | Test | mirrors content.service.ts | — |

**Admin Access**
| File | Type | Purpose | Depends on |
|---|---|---|---|
| src/schemas/adminAuth.schema.ts | Zod schema | loginSchema | — |
| src/services/adminAuth.service.ts | Service | authenticateAdmin, invalidateSession | prisma, bcrypt, jwt utils |
| src/controllers/adminAuth.controller.ts | Controller | login, logout, getMe handlers | adminAuth.service.ts |
| src/routes/adminAuth.routes.ts | Route | mounts /admin/auth/*; authMiddleware on logout + me only | adminAuth.controller.ts |
| tests/services/adminAuth.service.test.ts | Test | mirrors adminAuth.service.ts — includes generic-error-on-either-wrong-field case (FR-16) | — |

**Data Pipeline Management**
| File | Type | Purpose | Depends on |
|---|---|---|---|
| src/utils/dataSourceClients/viirs.client.ts | Util | wraps Google Earth Engine VIIRS pull | env config |
| src/utils/dataSourceClients/ghsl.client.ts | Util | wraps Google Earth Engine GHSL pull | env config |
| src/utils/dataSourceClients/rwi.client.ts | Util | reads static Meta RWI dataset | — |
| src/utils/dataSourceClients/gdelt.client.ts | Util | wraps GDELT API query | env config |
| src/schemas/adminPipeline.schema.ts | Zod schema | refreshParamsSchema, recomputeBodySchema, createWeightConfigSchema | — |
| src/services/adminPipeline.service.ts | Service | getSourcesWithHealth, startPipelineRun, getPipelineRuns, recomputeCompositeScores, getWeightConfigs, createAndActivateWeightConfig | prisma, dataSourceClients/* |
| src/controllers/adminPipeline.controller.ts | Controller | listSources, triggerRefresh, listRuns, triggerRecompute, listWeightConfigs, createWeightConfig handlers | adminPipeline.service.ts |
| src/routes/adminPipeline.routes.ts | Route | mounts /admin/pipeline/*; authMiddleware on every route | adminPipeline.controller.ts |
| tests/services/adminPipeline.service.test.ts | Test | mirrors adminPipeline.service.ts — includes weight-sum validation and GDELT-rejected-from-weights cases | — |

**Case Study Curation**
| File | Type | Purpose | Depends on |
|---|---|---|---|
| src/schemas/adminCaseStudies.schema.ts | Zod schema | createCaseStudySchema, updateCaseStudySchema, discoverBodySchema | — |
| src/services/adminCaseStudies.service.ts | Service | getAllCaseStudies, createCaseStudy, updateCaseStudy, deleteCaseStudy, searchCaseStudyCandidates | prisma, aiClient.ts |
| src/controllers/adminCaseStudies.controller.ts | Controller | listAllCaseStudies, createCaseStudy, updateCaseStudy, deleteCaseStudy, discoverCandidates handlers | adminCaseStudies.service.ts |
| src/routes/adminCaseStudies.routes.ts | Route | mounts /admin/case-studies/*; authMiddleware on every route | adminCaseStudies.controller.ts |
| tests/services/adminCaseStudies.service.test.ts | Test | mirrors adminCaseStudies.service.ts — includes date-order-warning (not rejection) and no-auto-publish-from-discovery cases | — |

**Cross-Cutting (Backend)**
| File | Type | Purpose | Depends on |
|---|---|---|---|
| src/utils/aiClient.ts | Util | wraps the LLM API — used for cell summaries, NL query parsing, and case-study discovery | env config (LLM API key) |
| prisma/seed.ts | Seed script | bootstraps the first (and only) Admin user, plus the four DataSource rows (VIIRS/GHSL/RWI/GDELT) | prisma |
| tests/utils/aiClient.test.ts | Test | mirrors aiClient.ts | — |

---

### 7.2 Schema/Config Changes

| File | Change |
|---|---|
| prisma/schema.prisma | Add all models + enums from Doc 4: Admin, DataSource, PipelineRun, GridCell, SignalValue, ScoreWeightConfig, SourceWeight, CompositeScoreSnapshot, CaseStudy; enums DataSourceKey, RunStatus, EvidenceTier |
| .env.example | Add GOOGLE_EARTH_ENGINE_CREDENTIALS, GDELT_API_KEY, LLM_API_KEY, ADMIN_SEED_EMAIL, ADMIN_SEED_PASSWORD |
| package.json | Add dependencies: an Earth Engine/Google API client, an LLM SDK, a lightweight geometry lib if needed for point-in-polygon checks (per Doc 4's no-PostGIS decision) |
| src/routes/index.ts | Register all 6 new routers: map, caseStudies, content, adminAuth, adminPipeline, adminCaseStudies |

---

### 7.3 File Creation Order

1. prisma/schema.prisma (migrate + generate first)
2. prisma/seed.ts
3. src/utils/aiClient.ts
4. src/utils/dataSourceClients/viirs.client.ts, ghsl.client.ts, rwi.client.ts, gdelt.client.ts
5. src/schemas/adminAuth.schema.ts
6. src/services/adminAuth.service.ts
7. src/controllers/adminAuth.controller.ts
8. src/routes/adminAuth.routes.ts
9. src/schemas/map.schema.ts
10. src/services/map.service.ts (depends on aiClient.ts)
11. src/controllers/map.controller.ts
12. src/routes/map.routes.ts
13. src/schemas/caseStudies.schema.ts
14. src/services/caseStudies.service.ts
15. src/controllers/caseStudies.controller.ts
16. src/routes/caseStudies.routes.ts
17. src/services/content.service.ts
18. src/controllers/content.controller.ts
19. src/routes/content.routes.ts
20. src/schemas/adminPipeline.schema.ts
21. src/services/adminPipeline.service.ts (depends on dataSourceClients/*)
22. src/controllers/adminPipeline.controller.ts
23. src/routes/adminPipeline.routes.ts
24. src/schemas/adminCaseStudies.schema.ts
25. src/services/adminCaseStudies.service.ts (depends on aiClient.ts)
26. src/controllers/adminCaseStudies.controller.ts
27. src/routes/adminCaseStudies.routes.ts
28. src/routes/index.ts (register everything above — last)
29. Test files, alongside each service per test-first discipline
