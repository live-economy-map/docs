## Project: Shadow Economy Map — Backend Test Documentation: Site Content & Onboarding
**Links back to:** [8-4. Function-Level Spec: Site Content & Onboarding]

Per the standing rule in `7a-backend-file-structure.md`: test-first, test file mirrors `src/` exactly under `tests/`, never co-located. Vitest — `describe`/`it`/`expect`, mocks via `vi.fn()`/`vi.mock()`, `beforeEach(() => vi.clearAllMocks())`.

---

### 9.1 Test File Map

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| src/services/content.service.ts | tests/services/content.service.test.ts | Unit (mocked Prisma via `vi.mock('@/lib/prisma')`) | ☐ |
| src/controllers/content.controller.ts | tests/controllers/content.controller.test.ts | Unit (mocked content.service via `vi.mock`) | ☐ |
| src/routes/content.routes.ts | tests/routes/content.routes.test.ts | Integration (supertest, mocked service layer) | ☐ |

No schema file, and therefore no schema test file — per Doc 8-4, neither endpoint in this feature accepts body, params, or query input.

---

### 9.2 Test Case Detail — content.service.test.ts

#### getLandingStats

| Case | Setup | Action | Expected result |
|---|---|---|---|
| At least one published case study, at least one successful pipeline run | mock `prisma.caseStudy.count` to resolve a nonzero number; mock the most-recent-successful-`PipelineRun` lookup to resolve a `completedAt` timestamp | call `getLandingStats()` | resolves `{ tagline, intro, highlightStats: { publishedCaseStudyCount: n, lastDataRefresh: "<timestamp>" } }` |
| No published case studies yet | mock `count` → `0` | call `getLandingStats()` | resolves `highlightStats.publishedCaseStudyCount: 0`, not an error |
| No successful pipeline run has ever completed (fresh deploy) | mock the most-recent-successful-run lookup → `null` | call `getLandingStats()` | resolves `highlightStats.lastDataRefresh: null` — per Doc 8-4, this is an explicit, expected edge case for a fresh deploy, not an error condition |
| Only counts published case studies | mock `count` call args spy | call `getLandingStats()` | assert `prisma.caseStudy.count` was called with `where: { isPublished: true }` — checked against call args, not just the returned number, so a query that accidentally counted drafts too would still fail the test |
| lastDataRefresh reflects only SUCCESS runs | mock the most-recent-run lookup to be scoped to `status: 'SUCCESS'` | call `getLandingStats()` | assert the underlying query filters on success status — a `FAILED` or `RUNNING` run must not surface as the "last data refresh," since that would misrepresent data freshness |

#### getMethodologyContent

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Returns all data sources, active and inactive | mock `prisma.dataSource.findMany()` to resolve 4 rows, one with `isActive: false` | call `getMethodologyContent()` | resolves `dataSources` containing all 4 — the inactive one is **not** filtered out; per Doc 8-4, the methodology page describes all sources ever used, unlike a pipeline dashboard that might show active-only. This is the specific distinction Doc 8-4 flags as easy to accidentally collapse — this test exists to catch that regression. |
| scoreExplanation/validationApproach/limitations are static, not DB-backed | spy on `prisma` calls | call `getMethodologyContent()` | assert no Prisma call was made for these three fields — only `dataSource.findMany()` should hit the database; a future change that accidentally moves this to a DB table without updating this test would be caught here |
| Empty DataSource table (should not occur in practice, but is not assumed impossible) | mock `findMany` → `[]` | call `getMethodologyContent()` | resolves `dataSources: []`, does not throw |

---

### 9.3 Test Case Detail — content.controller.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| getLandingContent delegates correctly | `vi.mock` content.service; `getLandingStats` resolves a landing content object | call controller (no input args) | `getLandingStats` called with no arguments; responds 200 via `SuccessResponse(200, "OK", result)` |
| getMethodologyContent delegates correctly | mock `getMethodologyContent` resolves a methodology content object | call controller | `getMethodologyContent` called with no arguments; responds 200 with `SuccessResponse(200, "OK", result)` |
| Controllers propagate unexpected service errors unchanged | mock either service function to throw a generic error | call the corresponding controller handler | error passed through via `asyncHandler`, not swallowed — even though neither service function is documented to throw under normal conditions, a controller must not silently eat an unexpected failure |

---

### 9.3 Test Case Detail — content.routes.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| GET /landing has no auth requirement | mock controller layer, no Authorization header | request `GET /content/landing` | 200, not 401 |
| GET /methodology has no auth requirement | mock controller layer, no Authorization header | request `GET /content/methodology` | 200, not 401 |
| Neither route has a validation middleware in its chain | mock controller layer | request either route with arbitrary query params | request reaches the controller mock unmodified — no `validate(...)` step to reject anything, since neither endpoint accepts input, per Doc 8-4 |

---

### 9.4 Coverage Honesty Check (per PR Steward, at review time)

- [ ] `getMethodologyContent`'s "includes inactive sources" case is actually asserted against a mock containing at least one `isActive: false` row — not just a mock of all-active rows that happens to pass either way
- [ ] `getLandingStats`'s `lastDataRefresh: null` fresh-deploy case is exercised with a genuine `null`-returning mock, not skipped as "obviously fine"
- [ ] The "static content is not DB-backed" assertion for `scoreExplanation`/`validationApproach`/`limitations` actually checks that no unexpected Prisma call was made — not just that the returned strings look reasonable
- [ ] No test hardcodes `publishedCaseStudyCount` or `dataSources` values that aren't actually derived from the mocked Prisma response

---

### 9.5 Out of Scope for Automated Testing (and why)

- **Actual wording/accuracy of the static methodology text** (`scoreExplanation`, `validationApproach`, `limitations`) — this is a content-review concern (Doc 2's "plain language accessible to a non-technical reader" NFR), checked via design/UX review, not a unit test assertion on string content.
- **Real Prisma query correctness against a live database** — same as every other feature, unit tests verify branching logic against mocked Prisma only.
- **Cache/staleness behavior on the frontend** (the 5-minute `staleTime` set in `useLandingContent`/`useMethodologyContent`, per frontend spec 3.3) — that's a frontend concern, not something this backend test suite has any visibility into.

---

**Next:** proceed to → [9-5. Backend Test Documentation: Admin Access]
