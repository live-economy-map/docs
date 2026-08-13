## Project: Shadow Economy Map — Backend Test Documentation: Case Study Curation
**Links back to:** [8-7. Function-Level Spec: Case Study Curation]

Per the standing rule in `7a-backend-file-structure.md`: test-first, test file mirrors `src/` exactly under `tests/`, never co-located. Vitest — `describe`/`it`/`expect`, mocks via `vi.fn()`/`vi.mock()`, `beforeEach(() => vi.clearAllMocks())`.

---

### 9.1 Test File Map

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| src/schemas/adminCaseStudies.schema.ts | tests/schemas/adminCaseStudies.schema.test.ts | Unit (schema parsing) | ☐ |
| src/services/adminCaseStudies.service.ts | tests/services/adminCaseStudies.service.test.ts | Unit (mocked Prisma via `vi.mock('@/lib/prisma')`, mocked `aiClient.ts`) | ☐ |
| src/controllers/adminCaseStudies.controller.ts | tests/controllers/adminCaseStudies.controller.test.ts | Unit (mocked adminCaseStudies.service via `vi.mock`) | ☐ |
| src/routes/adminCaseStudies.routes.ts | tests/routes/adminCaseStudies.routes.test.ts | Integration (supertest, mocked service layer + mocked `authMiddleware`) | ☐ |

---

### 9.2 Test Case Detail — adminCaseStudies.schema.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| createCaseStudySchema requires name, coordinates, evidenceDescription, both dates | — | parse `{ body: {} }` | validation fails |
| createCaseStudySchema accepts the minimal valid shape | — | parse `{ body: { name: "Bole Rd Expansion", latitude: 9.01, longitude: 38.78, evidenceDescription: "New commercial signage observed", scoreRiseDate: "2026-03-01", confirmedDate: "2026-05-01" } }` | passes; `isPublished` defaults to `false`; `gridCellId`/`evidenceUrl`/`evidenceTier`/`beforeImageUrl`/`afterImageUrl` all optional/undefined |
| createCaseStudySchema rejects an invalid evidenceUrl | — | parse body with `evidenceUrl: "not-a-url"` | validation fails |
| createCaseStudySchema rejects a malformed gridCellId | — | parse body with `gridCellId: "not-a-uuid"` | validation fails |
| createCaseStudySchema does not itself reject scoreRiseDate after confirmedDate | — | parse body with `scoreRiseDate: "2026-06-01", confirmedDate: "2026-01-01"` | passes — per Doc 8-7, this ordering is a service-level warning, not a schema-level rejection; a schema test asserting failure here would contradict the documented design |
| updateCaseStudySchema — all body fields optional | — | parse `{ body: {}, params: { caseStudyId: validUuid } }` | passes — confirms partial-update shape, distinct from create's required fields |
| updateCaseStudySchema requires a valid uuid caseStudyId param | — | parse `{ body: { name: "New name" }, params: { caseStudyId: "not-a-uuid" } }` | validation fails |
| discoverBodySchema requires a non-empty areaFocus | — | parse `{ body: { areaFocus: "" } }` | validation fails |
| discoverBodySchema accepts a valid areaFocus | — | parse `{ body: { areaFocus: "Bole subcity" } }` | passes |

`DELETE /admin/case-studies/:caseStudyId` has no body schema (per Doc 8-7) — no corresponding schema test case; a malformed `caseStudyId` is expected to 404 cleanly via the service.

---

### 9.2 Test Case Detail — adminCaseStudies.service.test.ts

#### getAllCaseStudies

| Case | Setup | Action | Expected result |
|---|---|---|---|
| No isPublished filter — returns both published and draft | mock `prisma.caseStudy.findMany` to resolve a mix of published/draft rows; spy on call args | call `getAllCaseStudies()` | assert `findMany` was called **without** an `isPublished` filter forced — distinct from the public `caseStudies.service.getPublishedCaseStudies`, which always forces `isPublished: true` (Doc 9-3); this is the key behavioral difference between the two services and deserves its own explicit assertion, not just "it returned some rows" |
| isPublished: true filter applied when requested | mock `findMany`; spy on call args | call `getAllCaseStudies(true)` | assert `findMany` was called with `where: { isPublished: true }` |
| isPublished: false filter applied when requested | mock `findMany`; spy on call args | call `getAllCaseStudies(false)` | assert `findMany` was called with `where: { isPublished: false }` |
| Empty result | mock `findMany` → `[]`; `count` → `0` | call `getAllCaseStudies()` | resolves `{ items: [], total: 0 }`, not an error |

#### createCaseStudy

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Valid input, no gridCellId, dates in normal order | mock `prisma.caseStudy.create` to resolve the new row | call `createCaseStudy(input, adminId)` | resolves `{ caseStudy, dateOrderWarning: false }` |
| Valid gridCellId provided and exists | mock a `GridCell` existence check to resolve a row; mock `create` | call `createCaseStudy({...input, gridCellId}, adminId)` | resolves successfully, `caseStudy.gridCellId` set |
| gridCellId provided but doesn't exist | mock the `GridCell` existence check → `null` | call `createCaseStudy({...input, gridCellId: badId}, adminId)` | throws `ApiError(404, "Grid cell not found")`; assert `prisma.caseStudy.create` was **not** called |
| scoreRiseDate after confirmedDate — warning, not rejection | mock `create` to resolve successfully with the out-of-order dates stored as-is | call `createCaseStudy({...input, scoreRiseDate: "2026-06-01", confirmedDate: "2026-01-01"}, adminId)` | resolves `{ caseStudy, dateOrderWarning: true }` — this is a **success case**, per Doc 8-7's explicit instruction ("explicitly covers the `dateOrderWarning: true` path as a success case, not an error case"); assert no `ApiError` is thrown and the row was actually created with the unusual dates intact, not silently corrected |
| Defaults isPublished to false when omitted | mock `create`; spy on call args | call `createCaseStudy(inputWithoutIsPublished, adminId)` | assert `create` was called with `isPublished: false` |
| createdById recorded | spy on `create` call args | call `createCaseStudy(input, adminId)` | assert the create payload includes `createdById: adminId` (or however Doc 4's schema names this relation) |

#### updateCaseStudy

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Valid partial update | mock `prisma.caseStudy.update` to resolve the updated row | call `updateCaseStudy(id, { evidenceUrl: "https://example.com/article" })` | resolves `{ caseStudy, dateOrderWarning: false }` |
| Case study doesn't exist | mock `update` to reject with a Prisma "record not found" error (or mock a preceding existence check → `null`, depending on implementation) | call `updateCaseStudy(badId, input)` | throws `ApiError(404, "Case study not found")` |
| Publishing a draft | mock `update` to resolve a row with `isPublished: true` | call `updateCaseStudy(id, { isPublished: true })` | resolves successfully — confirms this is the same function used for publishing, per Doc 8-7 ("including publishing"), not a separate endpoint |
| Update touches an unrelated field but pre-existing dates are already out of order | mock the existing row to already have `scoreRiseDate` after `confirmedDate`; mock `update` for an unrelated field (e.g., `evidenceUrl` only) to resolve the full merged record with the stale date order intact | call `updateCaseStudy(id, { evidenceUrl: "https://example.com/new-article" })` | resolves `dateOrderWarning: true` — the warning check re-evaluates the **resulting merged record**, not just the fields present in this particular request; this is the specific edge case Doc 8-7 flags as needing coverage, and skipping it would miss a real bug where the warning only fires on the request that introduces the bad ordering, not on subsequent unrelated edits |
| Update introduces a new gridCellId that doesn't exist | mock the `GridCell` existence check → `null` | call `updateCaseStudy(id, { gridCellId: badId })` | throws `ApiError(404, "Grid cell not found")` — same check as create, re-applied on update |

#### deleteCaseStudy

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Deletes an existing case study | mock `prisma.caseStudy.delete` to succeed | call `deleteCaseStudy(id)` | resolves void; assert `delete` was called with the matching `id` |
| Case study doesn't exist | mock `delete` to reject with a Prisma "record not found" error (or an existence check → `null`) | call `deleteCaseStudy(badId)` | throws `ApiError(404, "Case study not found")` |
| Hard delete, not soft delete | spy on the Prisma call used | call `deleteCaseStudy(id)` | assert `prisma.caseStudy.delete` was called, **not** `prisma.caseStudy.update` with an `isPublished`/`deletedAt`-style flag — per Doc 8-7's explicit note that this is a genuine hard delete, unlike the reference project's `Product.isPublished` soft-delete pattern; a test asserting only "the row is gone from a subsequent read" wouldn't catch an accidental soft-delete implementation, so this must check the actual Prisma method called |

#### searchCaseStudyCandidates

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Successful search with results | mock `aiClient`'s search/structured-output call (per Doc 9-1's assumed interface) to resolve a list of candidates | call `searchCaseStudyCandidates(areaFocus)` | resolves `{ candidates: [...] }`, each shaped per API spec 6.2 (`summary`, `sourceUrl`, `suggestedEvidenceTier`, `mentionedDate`) |
| No relevant results found — not an error | mock `aiClient` to resolve an empty candidate list | call `searchCaseStudyCandidates(areaFocus)` | resolves `{ candidates: [] }` — a normal 200-shaped success, per UC-15's alternate flow; does not throw |
| AI/search service unreachable or times out | mock `aiClient`'s call to reject | call `searchCaseStudyCandidates(areaFocus)` | throws `ApiError(400, "Discovery search is temporarily unavailable")` |
| Never writes to the database — regression guard | spy on every `prisma.caseStudy.*` method | call `searchCaseStudyCandidates(areaFocus)` | assert **none** of `create`/`update`/`delete`/`upsert` on `prisma.caseStudy` were called, regardless of how many candidates came back — this is the explicit regression guard Doc 8-7 calls for, protecting the no-auto-publish rule; this test should be written to fail loudly if a future change accidentally wires this function to persist anything |
| Candidates are not fabricated when the AI returns nothing usable | mock `aiClient` to resolve `null`/malformed output (same failure mode as `parseNaturalLanguageQuery`'s low-confidence case in Doc 9-2) | call `searchCaseStudyCandidates(areaFocus)` | resolves `{ candidates: [] }`, not a thrown error and not a fabricated placeholder candidate — mirrors the same graceful-degradation contract established for `aiClient.ts`'s structured-output path in Doc 9-1 |

---

### 9.3 Test Case Detail — adminCaseStudies.controller.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| listAllCaseStudies delegates correctly | `vi.mock` adminCaseStudies.service; `getAllCaseStudies` resolves `{items, page, limit, total}` | call controller with `req.query` | `getAllCaseStudies` called with `req.query.isPublished`, `req.query.page`, `req.query.limit`; responds 200 with `SuccessResponse(200, "OK", result)` |
| createCaseStudy responds with the normal message when no warning | mock `createCaseStudy` resolves `{caseStudy, dateOrderWarning: false}` | call controller with `req.body`, `req.user.id` | responds **201** with message `"Case study created"` |
| createCaseStudy responds with the warning message when dateOrderWarning is true | mock `createCaseStudy` resolves `{caseStudy, dateOrderWarning: true}` | call controller | responds 201 with the date-order-warning message from API spec 6.2 — controller branches on this field, same pattern as the reference project's cart capped-quantity branch (Doc 9-1's e-commerce example) |
| updateCaseStudy uses the same warning-branch pattern | mock `updateCaseStudy` resolves `{caseStudy, dateOrderWarning: true}` | call controller with `req.params.caseStudyId`, `req.body` | responds 200 with the warning message, same branching logic as create |
| updateCaseStudy uses the normal message when not warned | mock `updateCaseStudy` resolves `{caseStudy, dateOrderWarning: false}` | call controller | responds 200 with the normal (non-warning) message |
| deleteCaseStudy delegates correctly | mock `deleteCaseStudy` resolves void | call controller with `req.params.caseStudyId` | responds 200 with `SuccessResponse(200, "Case study deleted", {})` |
| discoverCandidates responds with "OK" when candidates are found | mock `searchCaseStudyCandidates` resolves `{candidates: [...]}` (non-empty) | call controller with `req.body.areaFocus` | responds 200 with message `"OK"` |
| discoverCandidates responds with the no-results message when candidates is empty | mock `searchCaseStudyCandidates` resolves `{candidates: []}` | call controller | responds 200 (not an error status) with message `"No relevant candidates found"` — branches on the `candidates` array length, same message-branching pattern used elsewhere in this doc set |
| discoverCandidates propagates the 400 unavailable error | mock `searchCaseStudyCandidates` to throw `ApiError(400, "Discovery search is temporarily unavailable")` | call controller | error passed through unchanged via `asyncHandler` |
| Controllers propagate the 404 grid-cell-not-found error unchanged | mock `createCaseStudy` or `updateCaseStudy` to throw `ApiError(404, "Grid cell not found")` | call the corresponding controller | error passed through, not swallowed or rewritten |

---

### 9.3 Test Case Detail — adminCaseStudies.routes.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Every route in this router requires auth | mock controller layer, no Authorization header | request each of GET /, POST /, PATCH /:caseStudyId, DELETE /:caseStudyId, POST /discover | all five return 401, controller mock never invoked |
| POST / validates body against createCaseStudySchema | valid auth, mock controller layer | request with an empty body | rejected by `validate(createCaseStudySchema)`, controller mock never called |
| PATCH /:caseStudyId validates the uuid param | valid auth, mock controller layer | request `PATCH /admin/case-studies/not-a-uuid` with a valid body | rejected by `validate(updateCaseStudySchema)`, controller mock never called |
| DELETE /:caseStudyId has no body schema | valid auth, mock controller layer | request `DELETE /admin/case-studies/some-id` with a malformed id | request reaches the controller mock (no `validate` middleware in the chain, per Doc 8-7) — a malformed id is handled by the service's 404, not blocked at the route layer |
| POST /discover validates areaFocus | valid auth, mock controller layer | request with `{ areaFocus: "" }` | rejected by `validate(discoverBodySchema)`, controller mock never called |

---

### 9.4 Coverage Honesty Check (per PR Steward, at review time)

- [ ] `getAllCaseStudies`'s "no forced isPublished filter" behavior is asserted against the Prisma mock's call args, not just against a mock dataset that happens to include both published and draft rows — this is the key distinction from the public case-studies service (Doc 9-3) and deserves a call-args-level assertion
- [ ] The `dateOrderWarning: true` path in both `createCaseStudy` and `updateCaseStudy` is tested as a genuine success (row created/updated, no thrown error) — not accidentally written as an error-case test that would contradict Doc 8-7's explicit instruction
- [ ] `updateCaseStudy`'s "warning re-evaluated against the merged record" case is actually implemented as its own test — not skipped as "basically the same as create's warning case," since the bug it guards against (warning only firing on the request that introduces the bad order, not on later unrelated edits) is specific to update
- [ ] `deleteCaseStudy`'s hard-delete assertion checks the actual Prisma method invoked (`delete`, not `update`) — not inferred from a subsequent read returning nothing, which a soft-delete would also satisfy
- [ ] `searchCaseStudyCandidates`'s no-database-writes regression guard is a real assertion against spied Prisma methods, present in the test file and not merely described in a comment
- [ ] The controller's message-branching tests (`dateOrderWarning`, `candidates.length === 0`) assert the actual response message string/shape, not just the status code, since the status code alone is identical in both branches

---

### 9.5 Out of Scope for Automated Testing (and why)

- **Real AI-assisted discovery quality** (whether `searchCaseStudyCandidates` genuinely finds relevant, non-fabricated leads) — unit tests mock `aiClient` entirely and can only verify the no-database-write guard and the empty/error/success branching; actual candidate relevance and the "never fabricate a plausible-sounding but unverified suggestion" prompt-design guarantee (Doc 8-7, UC-15) needs manual review of real model output, same category of exclusion as the AI summary quality check in Doc 9-2.
- **Real Prisma query correctness against a live database** — same as every other feature, unit tests verify branching logic against mocked Prisma only.
- **Evidence tier / evidence URL content verification** (whether a submitted `evidenceUrl` actually supports the claimed case study) — this is an editorial/human-review concern inherent to the admin curation workflow itself (UC-14), not something a test suite can or should verify; the backend only stores and returns what the admin provides.

---

**All backend test documentation complete.**
