## Project: Shadow Economy Map — Backend Function-Level Spec: Case Study Curation
Files covered: `src/schemas/adminCaseStudies.schema.ts` · `src/services/adminCaseStudies.service.ts` · `src/controllers/adminCaseStudies.controller.ts` · `src/routes/adminCaseStudies.routes.ts`

---

### src/schemas/adminCaseStudies.schema.ts (new)

| Schema | Shape |
|---|---|
| createCaseStudySchema | `z.object({ body: z.object({ name: z.string().min(1), latitude: z.number(), longitude: z.number(), gridCellId: z.string().uuid().optional(), evidenceDescription: z.string().min(1), evidenceUrl: z.string().url().optional(), evidenceTier: z.nativeEnum(EvidenceTier).optional(), scoreRiseDate: z.string().date(), confirmedDate: z.string().date(), beforeImageUrl: z.string().url().optional(), afterImageUrl: z.string().url().optional(), isPublished: z.boolean().optional().default(false) }) })` |
| updateCaseStudySchema | Same shape as above, all fields `.optional()` (partial update), plus `z.object({ params: z.object({ caseStudyId: z.string().uuid() }) })` |
| discoverBodySchema | `z.object({ body: z.object({ areaFocus: z.string().min(1) }) })` |

`DELETE /admin/case-studies/:caseStudyId` has no body schema — same rationale as elsewhere, malformed `:caseStudyId` 404s cleanly.

---

### src/services/adminCaseStudies.service.ts (new)

#### getAllCaseStudies

| Field | Detail |
|---|---|
| Signature | `getAllCaseStudies(isPublished?: boolean, page?: number, limit?: number): Promise<{ items: AdminCaseStudyDTO[]; page: number; limit: number; total: number }>` |
| Purpose | Admin-facing list, published and draft. |
| Inputs | Optional `isPublished` filter, pagination |
| Output | Shape per API spec 6.2 |
| Throws | None |
| Side effects | Read-only, no `isPublished: true` filter forced (unlike the public service) |
| Edge cases | Standard empty-result pattern. |

Test file: `tests/services/adminCaseStudies.service.test.ts`

#### createCaseStudy

| Field | Detail |
|---|---|
| Signature | `createCaseStudy(input: CaseStudyFormInput, createdById: string): Promise<{ caseStudy: CaseStudyDTO; dateOrderWarning: boolean }>` |
| Purpose | Creates a case study, defaulting to draft. |
| Inputs | Validated fields, `createdById` |
| Output | The created row, plus a `dateOrderWarning` flag |
| Throws | `ApiError(404, "Grid cell not found")` — `gridCellId` provided but doesn't exist |
| Side effects | `CaseStudy.create()` |
| Edge cases | `scoreRiseDate > confirmedDate` — **not rejected**, per the project's established expected-but-not-enforced pattern; the function still creates the row but returns `dateOrderWarning: true` so the controller surfaces a warning (API spec 6.2) rather than silently accepting an unusual case unnoticed. |

Test file: `tests/services/adminCaseStudies.service.test.ts` — explicitly covers the `dateOrderWarning: true` path as a success case, not an error case.

#### updateCaseStudy

| Field | Detail |
|---|---|
| Signature | `updateCaseStudy(id: string, input: Partial<CaseStudyFormInput>): Promise<{ caseStudy: CaseStudyDTO; dateOrderWarning: boolean }>` |
| Purpose | Partial update, including publishing (`isPublished: true`). |
| Inputs | `id`, partial fields |
| Output | Same shape as create |
| Throws | `ApiError(404, "Case study not found")` |
| Side effects | `CaseStudy.update()` |
| Edge cases | Date-order-warning check is re-evaluated against the *resulting* merged record, not just fields present in this request — an update that only touches `evidenceUrl` but leaves a pre-existing bad date order should still surface the warning. |

Test file: `tests/services/adminCaseStudies.service.test.ts`

#### deleteCaseStudy

| Field | Detail |
|---|---|
| Signature | `deleteCaseStudy(id: string): Promise<void>` |
| Purpose | Hard delete. |
| Inputs | `id` |
| Output | None |
| Throws | `ApiError(404, "Case study not found")` |
| Side effects | `CaseStudy.delete()` — a genuine hard delete, unlike the e-commerce template's `Product.isPublished` soft-delete pattern; case studies have no dependent downstream record that removal would corrupt, so soft-delete isn't needed (revisit if `CaseStudy` ever gains a dependent entity). |
| Edge cases | None beyond the 404. |

Test file: `tests/services/adminCaseStudies.service.test.ts`

#### searchCaseStudyCandidates

| Field | Detail |
|---|---|
| Signature | `searchCaseStudyCandidates(areaFocus: string): Promise<{ candidates: DiscoveryCandidateDTO[] }>` |
| Purpose | AI-assisted search for case-study leads — internal tool, never writes a `CaseStudy` row itself. |
| Inputs | `areaFocus` |
| Output | Shape per API spec 6.2 |
| Throws | `ApiError(400, "Discovery search is temporarily unavailable")` — AI/search service unreachable or times out |
| Side effects | External LLM/search call via `aiClient.ts`; **no database writes of any kind** — this function's only job is to surface suggestions for a human to review |
| Edge cases | No relevant results found → `candidates: []`, a normal 200, not an error — never fabricate a plausible-sounding but unverified suggestion (per UC-15's alternate flow; enforced via prompt design in `aiClient.ts`, not assumed). |

Test file: `tests/services/adminCaseStudies.service.test.ts` — explicitly asserts this function never calls any `prisma.caseStudy.*` write method, as a regression guard against the no-auto-publish rule ever being accidentally violated.

---

### src/controllers/adminCaseStudies.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| listAllCaseStudies | `adminCaseStudiesService.getAllCaseStudies(req.query.isPublished, req.query.page, req.query.limit)` | 200, `SuccessResponse(200, "OK", result)` |
| createCaseStudy | `adminCaseStudiesService.createCaseStudy(req.body, req.user.id)` | 201; message is `"Case study created"` normally, or the date-order-warning message from API spec 6.2 when `dateOrderWarning: true` — controller branches on this field, same pattern as the cart's capped-quantity branch in the e-commerce template |
| updateCaseStudy | `adminCaseStudiesService.updateCaseStudy(req.params.caseStudyId, req.body)` | 200, same warning-branch pattern as create |
| deleteCaseStudy | `adminCaseStudiesService.deleteCaseStudy(req.params.caseStudyId)` | 200, `SuccessResponse(200, "Case study deleted", {})` |
| discoverCandidates | `adminCaseStudiesService.searchCaseStudyCandidates(req.body.areaFocus)` | 200, `SuccessResponse(200, "OK" or "No relevant candidates found", { candidates })` |

---

### src/routes/adminCaseStudies.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | / | `authMiddleware` | listAllCaseStudies |
| POST | / | `authMiddleware, validate(createCaseStudySchema)` | createCaseStudy |
| PATCH | /:caseStudyId | `authMiddleware, validate(updateCaseStudySchema)` | updateCaseStudy |
| DELETE | /:caseStudyId | `authMiddleware` | deleteCaseStudy |
| POST | /discover | `authMiddleware, validate(discoverBodySchema)` | discoverCandidates |

`authMiddleware` applies to every route. Mounted at `/admin/case-studies`.

---

**All backend function-level specs complete.** Next: send the frontend template to continue with the mirrored frontend function-level specs.
