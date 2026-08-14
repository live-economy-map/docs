## Project: Shadow Economy Map — Frontend Function-Level Spec: Case Study Curation
Files covered: `src/pages/admin/CaseStudyCurationPage.tsx` · `src/components/admin-case-studies/*` · `src/hooks/useAdminCaseStudies.ts`

All routes and components in this feature sit behind `ProtectedRoute` + `DashboardLayout` (per frontend conventions 0.1). Same underlying `CaseStudy` entity as the public Case Studies feature (9-3), but this feature sees drafts too — never share query keys or cache entries between `QUERY_KEYS.CASE_STUDIES` (public) and `QUERY_KEYS.ADMIN_CASE_STUDIES` (this file), per API spec 6.1's deliberate router split.

---

### src/hooks/useAdminCaseStudies.ts (new — full block, per frontend spec 6.3)

#### useAdminCaseStudies

| Field | Detail |
|---|---|
| Signature | `useAdminCaseStudies(isPublished?: boolean, page = 1): UseQueryResult<{ items: AdminCaseStudy[]; page: number; limit: number; total: number }>` |
| Purpose | Wraps `GET /admin/case-studies`. |
| Query key | `[QUERY_KEYS.ADMIN_CASE_STUDIES, isPublished, page]` |
| Edge cases | `isPublished: undefined` returns both published and draft entries in one list — `CaseStudyTable`'s "all / published / draft" filter maps to this parameter, not to client-side filtering of a fetched-once list |
| Test file | `tests/hooks/useAdminCaseStudies.test.ts` |

#### useCreateCaseStudy

| Field | Detail |
|---|---|
| Signature | `useCreateCaseStudy(): UseMutationResult<{ id: string; name: string; isPublished: boolean }, AxiosError, CaseStudyFormInput>` |
| Purpose | Wraps `POST /admin/case-studies`. |
| Side effects | `onSuccess` invalidates `[QUERY_KEYS.ADMIN_CASE_STUDIES]` |
| Edge cases | A `201` response whose `message` contains the date-order warning text (per API spec 6.2 — `scoreRiseDate` after `confirmedDate`) is **still a success**, not a mutation error — `CaseStudyForm` must inspect the response `message` string on success and, if it matches the warning case, show an inline warning banner rather than closing the form silently, per frontend spec 6.4. A `404 "Grid cell not found"` (invalid `gridCellId`) is a genuine mutation error, attributable to the `gridCellId` field specifically |
| Test file | `tests/hooks/useAdminCaseStudies.test.ts` — includes the date-order-warning-is-still-success case |

#### useUpdateCaseStudy

| Field | Detail |
|---|---|
| Signature | `useUpdateCaseStudy(): UseMutationResult<{ id: string; name: string; isPublished: boolean }, AxiosError, { id: string; input: Partial<CaseStudyFormInput> }>` |
| Purpose | Wraps `PATCH /admin/case-studies/:caseStudyId`. |
| Side effects | `onSuccess` invalidates `[QUERY_KEYS.ADMIN_CASE_STUDIES]` — this is also the mutation used to publish a draft (`input: { isPublished: true }`), there is no separate "publish" endpoint or hook, per API spec 6.2 |
| Edge cases | The same date-order-warning-in-message pattern as `useCreateCaseStudy` applies here too, since edits can reintroduce the same inconsistency; `CaseStudyForm` handles both mutations through one shared response-inspection code path rather than duplicating the check |
| Test file | `tests/hooks/useAdminCaseStudies.test.ts` |

#### useDeleteCaseStudy

| Field | Detail |
|---|---|
| Signature | `useDeleteCaseStudy(): UseMutationResult<void, AxiosError, string>` (variable: `id`) |
| Purpose | Wraps `DELETE /admin/case-studies/:caseStudyId`. |
| Side effects | `onSuccess` invalidates `[QUERY_KEYS.ADMIN_CASE_STUDIES]` |
| Edge cases | `CaseStudyTable`'s delete action requires an explicit confirm step (e.g., a confirm dialog) before calling `.mutate(id)` — deletion is irreversible and removes the entry from the public map/list immediately per UC-14's remove flow, so it must not be a single accidental click |
| Test file | `tests/hooks/useAdminCaseStudies.test.ts` |

#### useDiscoverCandidates

| Field | Detail |
|---|---|
| Signature | `useDiscoverCandidates(): UseMutationResult<{ candidates: DiscoveryCandidate[] }, AxiosError, string>` (variable: `areaFocus`) |
| Purpose | Wraps `POST /admin/case-studies/discover`. |
| Side effects | No query invalidation — results are consumed directly by `DiscoveryPanel`, never written to the `ADMIN_CASE_STUDIES` cache, since a candidate is not a `CaseStudy` until an admin explicitly creates one (per UC-15) |
| Edge cases | An empty `candidates: []` response (with `message: "No relevant candidates found"`) is a **success**, rendered as an in-panel "no candidates found" message — not a mutation error. A `400 "Discovery search is temporarily unavailable"` (AI/search service down) **is** a genuine mutation error, shown distinctly from the empty-results case for the same reason as `MapSearchBar`'s equivalent distinction in frontend spec 9-2 |
| Test file | `tests/hooks/useAdminCaseStudies.test.ts` — includes the discovery-never-auto-publishes case (per 7b file structure §7.5) and the empty-results-is-not-an-error case |

---

### src/components/admin-case-studies/CaseStudyTable.tsx (new — full block)

| Field | Detail |
|---|---|
| Props | `{ onEdit: (caseStudy: AdminCaseStudy) => void }` |
| Local state | `publishedFilter: boolean \| undefined` ('all' / 'published' / 'draft'), `page: number` |
| Behavior | `useAdminCaseStudies(publishedFilter, page)`; renders `name`, `isPublished` badge, `evidenceTier`, `createdAt` per row; each row has an "Edit" action (calls `onEdit(caseStudy)`, which the parent page uses to open `CaseStudyForm` pre-filled) and a "Delete" action wired to `useDeleteCaseStudy` behind a confirm step |
| Edge cases | `items: []` for the current filter — renders an empty-state row/message, not a blocked table (e.g., "no draft case studies" reads differently from "no case studies at all," so the empty message should reflect the active filter) |
| Test file | `tests/components/admin-case-studies/CaseStudyTable.test.tsx` |

### src/components/admin-case-studies/CaseStudyForm.tsx (new — full block)

| Field | Detail |
|---|---|
| Props | `{ mode: 'create' \| 'edit'; initialValues?: Partial<CaseStudyFormInput>; caseStudyId?: string; onSuccess?: () => void }` |
| Local state | React Hook Form state for all `CaseStudyFormInput` fields; `initialValues` is used both for editing an existing entry and for pre-filling from a `DiscoveryPanel` candidate (mapped to `evidenceDescription`/`evidenceUrl`/`evidenceTier`/`scoreRiseDate`≈`mentionedDate` as a starting point, never auto-submitted) |
| Behavior | React Hook Form + Zod, per frontend spec 6.4. Client-side schema mirrors the backend's `createCaseStudySchema`/`updateCaseStudySchema` (required `name`/`latitude`/`longitude`/`evidenceDescription`/`scoreRiseDate`/`confirmedDate`; optional `gridCellId`/`evidenceUrl`/`evidenceTier`/`beforeImageUrl`/`afterImageUrl`/`isPublished`). On submit, calls `useCreateCaseStudy()` (mode `create`) or `useUpdateCaseStudy()` (mode `edit`, using `caseStudyId`). |
| Edge cases | On a successful response whose `message` contains the `scoreRiseDate`-after-`confirmedDate` warning (per API spec 6.2), renders a **visible inline warning banner inside the form** — not a toast — so it's seen before the admin navigates away, per frontend spec 6.4's explicit requirement. A `404 "Grid cell not found"` attaches to the `gridCellId` field specifically, not a generic form banner, since it's a single-field problem the admin can directly fix |
| Test file | `tests/components/admin-case-studies/CaseStudyForm.test.tsx` — includes the date-order-warning-renders-as-banner case |

### src/components/admin-case-studies/DiscoveryPanel.tsx (new — full block)

| Field | Detail |
|---|---|
| Props | `{ onUseCandidate: (candidate: DiscoveryCandidate) => void }` |
| Local state | `areaFocus: string` |
| Behavior | On submit, calls `useDiscoverCandidates().mutate(areaFocus)`; renders each returned candidate (`summary`, `sourceUrl` as an outbound link, `suggestedEvidenceTier`, `mentionedDate`) with a "Use this" button that calls `onUseCandidate(candidate)` — the parent page (`CaseStudyCurationPage`) wires this to open `CaseStudyForm` in create mode with `initialValues` derived from the candidate |
| Edge cases | **"Use this" never itself calls `useCreateCaseStudy` or otherwise publishes anything** — per frontend spec 6.5's important UX rule, it only pre-fills the form; the admin must still review and explicitly submit `CaseStudyForm`. This is a hard constraint on this component's implementation, not an incidental detail: a future "optimize this into a one-click publish" change would violate UC-15's postcondition and must not be made here. An empty-results response renders "No relevant candidates found for this area" inline, distinct from the temporarily-unavailable mutation-error case (see `useDiscoverCandidates` above) |
| Test file | `tests/components/admin-case-studies/DiscoveryPanel.test.tsx` — includes an explicit assertion that clicking "Use this" does not call any create/update/publish mutation |

---

### src/pages/admin/CaseStudyCurationPage.tsx (new)

| Field | Detail |
|---|---|
| Purpose | CRUD plus AI-assisted discovery for case studies, in one admin page (UC-14, UC-15). |
| Route guard + layout | `ProtectedRoute`, `DashboardLayout` |
| Local state | `formOpen: boolean`, `formMode: 'create' \| 'edit'`, `editingCaseStudy: AdminCaseStudy \| null`, `formInitialValues: Partial<CaseStudyFormInput> \| undefined` |
| Behavior | 1. Renders `CaseStudyTable` with `onEdit` opening the form in `edit` mode, pre-filled from the selected row (a follow-up detail fetch may be needed if `CaseStudyTable`'s summary shape lacks fields `CaseStudyForm` needs — e.g., coordinates/evidence text not present in `AdminCaseStudy`; the page's `onEdit` handler is responsible for supplying complete `initialValues` to the form, not `CaseStudyTable` itself). 2. Renders `DiscoveryPanel` with `onUseCandidate` opening the form in `create` mode, pre-filled from the candidate. 3. Renders `CaseStudyForm` conditionally when `formOpen`, with `onSuccess` closing the form and relying on the hooks' own cache invalidation to refresh `CaseStudyTable`. |
| Components | CaseStudyTable, CaseStudyForm, DiscoveryPanel |

**States:** loading (table skeleton) · error (inline + retry, per section) · empty (table shows filtered empty-state; discovery panel shows no-candidates state) · success

---

**All frontend function-level spec files complete.**
