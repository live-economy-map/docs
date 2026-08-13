## Project: Shadow Economy Map — Backend Test Documentation: Case Studies & Validation
**Links back to:** [8-3. Function-Level Spec: Case Studies & Validation]

Per the standing rule in `7a-backend-file-structure.md`: test-first, test file mirrors `src/` exactly under `tests/`, never co-located. Vitest — `describe`/`it`/`expect`, mocks via `vi.fn()`/`vi.mock()`, `beforeEach(() => vi.clearAllMocks())`.

---

### 9.1 Test File Map

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| src/schemas/caseStudies.schema.ts | tests/schemas/caseStudies.schema.test.ts | Unit (schema parsing) | ☐ |
| src/services/caseStudies.service.ts | tests/services/caseStudies.service.test.ts | Unit (mocked Prisma via `vi.mock('@/lib/prisma')`) | ☐ |
| src/controllers/caseStudies.controller.ts | tests/controllers/caseStudies.controller.test.ts | Unit (mocked caseStudies.service via `vi.mock`) | ☐ |
| src/routes/caseStudies.routes.ts | tests/routes/caseStudies.routes.test.ts | Integration (supertest, mocked service layer) | ☐ |

---

### 9.2 Test Case Detail — caseStudies.schema.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Defaults page/limit when omitted | — | parse `{ query: {} }` | `result.query.page === 1`; `result.query.limit === 20` |
| Coerces string page/limit to numbers | — | parse `{ query: { page: "2", limit: "10" } }` | `result.query.page === 2` (number); `result.query.limit === 10` |
| Rejects limit above 100 | — | parse `{ query: { limit: "101" } }` | validation fails |
| Rejects non-positive page | — | parse `{ query: { page: "0" } }` | validation fails |

`GET /case-studies/:caseStudyId` has no schema (per Doc 8-3) — a malformed `caseStudyId` is expected to 404 cleanly via the service's Prisma lookup, so there is no corresponding schema test case for it.

---

### 9.2 Test Case Detail — caseStudies.service.test.ts

#### getPublishedCaseStudies

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Published-only filter applied | mock `prisma.caseStudy.findMany` to resolve an array; spy on call args | call `getPublishedCaseStudies()` | assert `findMany` was called with `where: { isPublished: true }` — checked against the mock's call args, not just the return value, to confirm the filter is actually sent to Prisma and not just true by coincidence of the mock data |
| Default pagination | mock `findMany` and a `count` resolving `total` | call `getPublishedCaseStudies()` | resolves `{ items: [...], page: 1, limit: 20, total }` |
| Explicit pagination passed through | mock `findMany`/`count` | call `getPublishedCaseStudies(2, 10)` | resolves `{ page: 2, limit: 10, ... }`; assert `findMany` was called with the matching `skip`/`take` |
| Zero published case studies | mock `findMany` → `[]`; mock `count` → `0` | call `getPublishedCaseStudies()` | resolves `{ items: [], page: 1, limit: 20, total: 0 }`, does not throw — per UC-08's alternate flow |
| Page beyond the last page | mock `findMany` → `[]`; mock `count` → nonzero (e.g., 3) | call `getPublishedCaseStudies(99)` | resolves `{ items: [], total: 3, page: 99 }`, not an error |

#### getPublishedCaseStudyById

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Case study exists and is published | mock lookup to resolve a row with `isPublished: true` | call `getPublishedCaseStudyById(id)` | resolves the full `CaseStudyDetailDTO`, including `evidenceDescription`, `evidenceUrl`, `beforeImageUrl`/`afterImageUrl` (or `null`s) |
| Case study doesn't exist | mock lookup → `null` | call `getPublishedCaseStudyById(badId)` | throws `ApiError(404, "Case study not found")` |
| Case study exists but isPublished is false | mock lookup to resolve a row with `isPublished: false` | call `getPublishedCaseStudyById(id)` | throws the exact same `ApiError(404, "Case study not found")` — assert identical status/message to the "doesn't exist" case; this is the case Doc 8-3 flags explicitly ("never reveals draft entries") |
| Both before/after image URLs are null | mock lookup to resolve a row with `beforeImageUrl: null, afterImageUrl: null` | call `getPublishedCaseStudyById(id)` | resolves successfully with both fields `null`, unchanged — the service does not substitute a placeholder image URL; per Doc 8-3, that decision belongs to the frontend |
| Only one of before/after is present | mock lookup to resolve a row with `beforeImageUrl` set, `afterImageUrl: null` | call `getPublishedCaseStudyById(id)` | resolves with both fields exactly as stored — the service does not enforce "both or neither" itself; that enforcement lives in the frontend's `CaseStudyDetailPanel` (per frontend spec 2.4), not here |

---

### 9.3 Test Case Detail — caseStudies.controller.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| listCaseStudies delegates correctly | `vi.mock` caseStudies.service; `getPublishedCaseStudies` resolves `{items, page, limit, total}` | call controller with `req.query` | `getPublishedCaseStudies` called with `req.query.page`, `req.query.limit`; responds 200 via `SuccessResponse(200, "OK", result)` |
| getCaseStudyById delegates correctly | mock `getPublishedCaseStudyById` resolves a case study | call controller with `req.params.caseStudyId` | `getPublishedCaseStudyById` called with `req.params.caseStudyId`; responds 200 with `SuccessResponse(200, "OK", caseStudy)` |
| getCaseStudyById propagates 404 | mock `getPublishedCaseStudyById` to throw `ApiError(404, "Case study not found")` | call controller | error passed through to error-handling middleware via `asyncHandler`, not swallowed or rewritten |

---

### 9.3 Test Case Detail — caseStudies.routes.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| GET / has no auth requirement | mock controller layer, no Authorization header | request `GET /case-studies` | 200, not 401 — confirms fully public |
| GET / validates pagination params | mock controller layer | request `GET /case-studies?limit=999` | rejected by `validate(listCaseStudiesQuerySchema)` (limit > 100), controller mock never called |
| GET /:caseStudyId has no auth requirement | mock controller layer, no Authorization header | request `GET /case-studies/some-id` | 200, not 401 |
| GET /:caseStudyId skips schema validation entirely | mock controller layer | request `GET /case-studies/not-a-valid-uuid` | request reaches the controller mock (no `validate` middleware in the chain, per Doc 8-3); a malformed id is expected to be handled by the service's 404, not blocked at the route layer |

---

### 9.4 Coverage Honesty Check (per PR Steward, at review time)

- [ ] The unpublished-vs-nonexistent 404 case asserts the *identical* `ApiError` (status and message), not just "both throw a 404" — this is the exact case Doc 8-3 calls out as needing explicit coverage
- [ ] The `findMany` published-only filter test asserts against the Prisma mock's call args, not just the returned data — a service that forgot the `where: { isPublished: true }` clause but happened to be tested with only-published mock data would otherwise pass incorrectly
- [ ] The before/after image nullability tests cover all three states (both present, one present, both null) — not just the "both present" happy path
- [ ] No test asserts against a hardcoded `total`/`page` value that isn't actually derived from the mocked Prisma `count`/`findMany` response

---

### 9.5 Out of Scope for Automated Testing (and why)

- **Real Prisma query correctness against a live database** (e.g., whether the `isPublished: true` filter combined with pagination actually performs correctly at scale) — unit tests verify logic branching against mocked Prisma only; a real-database integration pass is tracked separately, same as every other feature's service tests.
- **Before/after image loading/rendering** — this is the frontend `BeforeAfterSlider` component's concern (frontend spec 9-3); the backend service only stores and returns URLs, it never fetches or validates the images themselves.
- **Pagination performance at scale** — unit tests verify the pagination shape and edge cases (empty page, page beyond last), not query performance under a large `CaseStudy` table (which, per V1's scope, is expected to stay in the single digits to low tens of rows anyway).

---

**Next:** proceed to → [9-4. Backend Test Documentation: Site Content & Onboarding]
