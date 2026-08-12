## Project: Shadow Economy Map — Backend Function-Level Spec: Case Studies & Validation
Files covered: `src/schemas/caseStudies.schema.ts` · `src/services/caseStudies.service.ts` · `src/controllers/caseStudies.controller.ts` · `src/routes/caseStudies.routes.ts`

---

### src/schemas/caseStudies.schema.ts (new)

| Schema | Shape |
|---|---|
| listCaseStudiesQuerySchema | `z.object({ query: z.object({ page: z.coerce.number().int().positive().optional().default(1), limit: z.coerce.number().int().positive().max(100).optional().default(20) }) })` |

`GET /case-studies/:caseStudyId` has no schema — same rationale as the e-commerce template's product-detail route: a malformed `:caseStudyId` just 404s cleanly via Prisma's lookup failure, no separate uuid-format check needed.

---

### src/services/caseStudies.service.ts (new)

#### getPublishedCaseStudies

| Field | Detail |
|---|---|
| Signature | `getPublishedCaseStudies(page?: number, limit?: number): Promise<{ items: CaseStudySummaryDTO[]; page: number; limit: number; total: number }>` |
| Purpose | Paginated, published-only summary list — powers both map pins and the list page. |
| Inputs | `page` (default 1), `limit` (default 20) |
| Output | Shape per API spec 2.2 |
| Throws | None |
| Side effects | Read-only `CaseStudy.findMany({ where: { isPublished: true } })` |
| Edge cases | Zero published case studies — `items: []`, `total: 0`, not an error. `page` beyond the last page — same, not an error. |

Test file: `tests/services/caseStudies.service.test.ts`

#### getPublishedCaseStudyById

| Field | Detail |
|---|---|
| Signature | `getPublishedCaseStudyById(id: string): Promise<CaseStudyDetailDTO>` |
| Purpose | Full evidence detail for one published case study. |
| Inputs | `id` |
| Output | Shape per API spec 2.2 |
| Throws | `ApiError(404, "Case study not found")` — `id` doesn't exist, or exists but `isPublished: false`, identical either way (never reveals draft entries) |
| Side effects | Read-only |
| Edge cases | `beforeImageUrl`/`afterImageUrl` both `null` — return as-is, don't substitute a placeholder; the frontend decides how to render that state (per frontend spec 2.4). |

Test file: `tests/services/caseStudies.service.test.ts` — explicitly covers the unpublished-returns-404-identically-to-nonexistent case.

---

### src/controllers/caseStudies.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| listCaseStudies | `caseStudiesService.getPublishedCaseStudies(req.query.page, req.query.limit)` | 200, `SuccessResponse(200, "OK", result)` |
| getCaseStudyById | `caseStudiesService.getPublishedCaseStudyById(req.params.caseStudyId)` | 200, `SuccessResponse(200, "OK", caseStudy)` |

---

### src/routes/caseStudies.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| GET | / | `validate(listCaseStudiesQuerySchema)` | listCaseStudies |
| GET | /:caseStudyId | — | getCaseStudyById |

No `authMiddleware` — fully public. Mounted at `/case-studies`.

---

**Next:** proceed to → [8-4. Backend: Site Content & Onboarding]
