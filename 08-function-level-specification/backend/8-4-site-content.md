## Project: Shadow Economy Map — Backend Function-Level Spec: Site Content & Onboarding
Files covered: `src/services/content.service.ts` · `src/controllers/content.controller.ts` · `src/routes/content.routes.ts`

No schema file — neither endpoint in this feature accepts body, params, or query input.

---

### src/services/content.service.ts (new)

#### getLandingStats

| Field | Detail |
|---|---|
| Signature | `getLandingStats(): Promise<LandingContentDTO>` |
| Purpose | Assembles landing-page copy plus live stats (published case study count, last data refresh timestamp). |
| Inputs | None |
| Output | Shape per API spec 3.2 |
| Throws | None |
| Side effects | Read-only: `CaseStudy.count({ where: { isPublished: true } })` and the most recent successful `PipelineRun.completedAt` across all sources |
| Edge cases | No successful `PipelineRun` exists yet (fresh deploy, before first seed/refresh) → `lastDataRefresh: null`, not an error; frontend handles the null case per frontend spec 3.4. |

Test file: `tests/services/content.service.test.ts`

#### getMethodologyContent

| Field | Detail |
|---|---|
| Signature | `getMethodologyContent(): Promise<MethodologyContentDTO>` |
| Purpose | Assembles the methodology page: score explanation, data source list (from the `DataSource` table), validation approach, limitations. |
| Inputs | None |
| Output | Shape per API spec 3.2 |
| Throws | None |
| Side effects | Read-only `DataSource.findMany()` for the `dataSources` array; `scoreExplanation`, `validationApproach`, and `limitations` are static strings maintained in this service (not DB-backed — they change only via a code deploy, matching their nature as fixed methodology text, not operational data) |
| Edge cases | `DataSource.isActive: false` rows — still included here (methodology page describes all sources ever used, active or not), unlike pipeline dashboards which may want to filter to active-only; confirm this distinction doesn't get accidentally collapsed during implementation. |

Test file: `tests/services/content.service.test.ts`

---

### src/controllers/content.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| getLandingContent | `contentService.getLandingStats()` | 200, `SuccessResponse(200, "OK", result)` |
| getMethodologyContent | `contentService.getMethodologyContent()` | 200, `SuccessResponse(200, "OK", result)` |

---

### src/routes/content.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | /landing | — | getLandingContent |
| GET | /methodology | — | getMethodologyContent |

No `authMiddleware` — fully public. Mounted at `/content`.

---

**Next:** proceed to → [8-5. Backend: Admin Access]
