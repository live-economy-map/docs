## Project: Shadow Economy Map — Frontend Test Documentation: Case Study Curation
**Links back to:** [9-7. Frontend Function-Level Spec: Case Study Curation], [9-1. Frontend Test Documentation: Shared/Config], [9-3. Frontend Test Documentation: Case Studies & Validation]

Per the Frontend Testing Guide: hooks mock `@/lib/axios`; pages/components mock this feature's own hooks (`@/hooks/useAdminCaseStudies`). Every hook/page in this module is Admin-authenticated (per API spec 5.1) — none of these tests need to simulate the auth guard itself (that's 9-1's `ProtectedRoute` coverage); they assume an authenticated context, consistent with 9-6's equivalent note for the pipeline-management module. This module shares the `CaseStudy` entity with the public Case Studies feature (9-3) but never its query keys or cache entries (`QUERY_KEYS.ADMIN_CASE_STUDIES` vs. `QUERY_KEYS.CASE_STUDIES`, per API spec 6.1's router split) — no test in this file should assert against or invalidate the public feature's cache. This is a High-Scrutiny module (Guide Section 12): the discovery-never-auto-publishes guarantee (UC-15) and the date-order-warning-is-still-success branch (API spec 6.2) are both easy to get subtly wrong, same category of risk as 9-5's generic-401 discipline.

---

### 9.1 Test File Map

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| `src/hooks/useAdminCaseStudies.ts` | `tests/hooks/useAdminCaseStudies.test.ts` | Unit (`renderHook`, mocked `@/lib/axios`) | ☐ |
| `src/components/admin-case-studies/CaseStudyTable.tsx` | `tests/components/admin-case-studies/CaseStudyTable.test.tsx` | Component (RTL, mocked `useAdminCaseStudies` + `useDeleteCaseStudy`) | ☐ |
| `src/components/admin-case-studies/CaseStudyForm.tsx` | `tests/components/admin-case-studies/CaseStudyForm.test.tsx` | Component (RTL, mocked `useCreateCaseStudy` + `useUpdateCaseStudy`) | ☐ |
| `src/components/admin-case-studies/DiscoveryPanel.tsx` | `tests/components/admin-case-studies/DiscoveryPanel.test.tsx` | Component (RTL, mocked `useDiscoverCandidates`) | ☐ |
| `src/pages/admin/CaseStudyCurationPage.tsx` | `tests/pages/admin/CaseStudyCurationPage.test.tsx` | Component (RTL, mocked `CaseStudyTable`/`CaseStudyForm`/`DiscoveryPanel` as child components) | ☐ |

---

### 9.2 Test Case Detail — useAdminCaseStudies.test.ts

#### useAdminCaseStudies

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Calls GET /admin/case-studies with isPublished and page | mocked `api.get` resolves `{items, page, limit, total}` | `renderHook(() => useAdminCaseStudies(true, 2))` | `api.get` called with `('/admin/case-studies', {params: {isPublished: true, page: 2}})` |
| isPublished omitted returns both published and draft — not client-filtered | mocked `api.get` resolves a mix of published/draft rows; spy on call args | `renderHook(() => useAdminCaseStudies(undefined, 1))` | `api.get` called with `params: {isPublished: undefined, page: 1}` — the "all" filter is a request-level param sent to the server, not a client-side `.filter()` over a fetched-once list, per Doc 9-7's explicit distinction |
| Query key includes both isPublished and page | inspect cache | render with given args | cache entry keyed exactly `[QUERY_KEYS.ADMIN_CASE_STUDIES, isPublished, page]` — distinct key per filter/page combination, so switching the table's filter doesn't serve stale cached data from a different filter |
| Empty result for the current filter | mocked `api.get` resolves `{items: [], page: 1, limit: 20, total: 0}` | render, await | resolves cleanly with `items: []`, `isError` stays false |

#### useCreateCaseStudy

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Calls POST /admin/case-studies with form input | mocked `api.post` resolves `{data: {id, name, isPublished}}` | `renderHook(() => useCreateCaseStudy())`, `.mutate(input)` | `api.post` called with `('/admin/case-studies', input)` |
| On success, invalidates the admin case studies cache | mock `queryClient.invalidateQueries` | `.mutate(input)`, await success | `invalidateQueries` called with `{queryKey: [QUERY_KEYS.ADMIN_CASE_STUDIES]}` |
| Date-order-warning response is a genuine success, not a mutation error | mocked `api.post` resolves 201 with a `message` containing the date-order warning text (per API spec 6.2) | `.mutate(input)`, await | `result.current.isSuccess === true`, `result.current.isError === false` — the mutation itself does not branch on message content; per Doc 9-7, inspecting `message` to show the warning banner is `CaseStudyForm`'s job, not this hook's |
| 404 grid-cell-not-found propagates as a genuine mutation error | mocked `api.post` rejects with `{response: {status: 404, data: {message: 'Grid cell not found'}}}` | `.mutate({...input, gridCellId: badId})` | `result.current.isError === true`; `result.current.error.response.data.message === 'Grid cell not found'`, passed through unmodified |

#### useUpdateCaseStudy

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Calls PATCH /admin/case-studies/:id with a partial body | mocked `api.patch` resolves | `renderHook(() => useUpdateCaseStudy())`, `.mutate({id, input: {evidenceUrl: 'https://example.com/article'}})` | `api.patch` called with `('/admin/case-studies/' + id, {evidenceUrl: 'https://example.com/article'})` |
| On success, invalidates the admin case studies cache | mock `queryClient.invalidateQueries` | `.mutate({id, input})`, await success | `invalidateQueries` called with `{queryKey: [QUERY_KEYS.ADMIN_CASE_STUDIES]}` |
| Doubles as the publish action — no separate publish hook | mocked `api.patch` resolves; spy on call args | `.mutate({id, input: {isPublished: true}})` | `api.patch` called with `isPublished: true` in the body — confirms there is no dedicated `usePublishCaseStudy` hook, per Doc 9-7's explicit "no separate publish endpoint" note; a test suite that skipped this because it "looks like a normal update" would leave this documented design decision unverified |
| Date-order-warning response is a genuine success on update too | mocked `api.patch` resolves 200 with the date-order warning message | `.mutate({id, input: {scoreRiseDate: '2026-06-01', confirmedDate: '2026-01-01'}})` | `result.current.isSuccess === true` — same non-branching contract as `useCreateCaseStudy`, exercised here as its own test since edits can reintroduce the inconsistency independently of creation (Doc 9-7) |
| Case study not found propagates as an error | mocked `api.patch` rejects with a 404 AxiosError | `.mutate({id: badId, input})` | `result.current.isError === true` |

#### useDeleteCaseStudy

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Calls DELETE /admin/case-studies/:id | mocked `api.delete` resolves | `renderHook(() => useDeleteCaseStudy())`, `.mutate(id)` | `api.delete` called with `('/admin/case-studies/' + id)` |
| On success, invalidates the admin case studies cache | mock `queryClient.invalidateQueries` | `.mutate(id)`, await success | `invalidateQueries` called with `{queryKey: [QUERY_KEYS.ADMIN_CASE_STUDIES]}` |
| Deletion failure surfaces as an error, cache not invalidated | mocked `api.delete` rejects | `.mutate(id)` | `result.current.isError === true`; `invalidateQueries` NOT called |

#### useDiscoverCandidates

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Calls POST /admin/case-studies/discover with areaFocus | mocked `api.post` resolves `{data: {candidates: [...]}}` | `renderHook(() => useDiscoverCandidates())`, `.mutate(areaFocus)` | `api.post` called with `('/admin/case-studies/discover', {areaFocus})` |
| Empty candidates is a success, not an error | mocked `api.post` resolves 200 with `{candidates: []}` | `.mutate(areaFocus)`, await | `result.current.isSuccess === true`; `result.current.data.candidates` is `[]` — rendering the in-panel "no candidates found" message is `DiscoveryPanel`'s job, per Doc 9-7 |
| Discovery service unavailable propagates as an error | mocked `api.post` rejects with `{response: {status: 400, data: {message: 'Discovery search is temporarily unavailable'}}}` | `.mutate(areaFocus)` | `result.current.isError === true`; message passed through unmodified, distinguishable from the empty-results success case above |
| Never invalidates or writes to the admin case studies cache — regression guard | mock `queryClient.invalidateQueries`; mocked `api.post` resolves a non-empty `candidates` list | `.mutate(areaFocus)`, await success | `invalidateQueries` NOT called with any `ADMIN_CASE_STUDIES`-keyed query — per Doc 7b §7.5's explicit "discovery-never-auto-publishes" standing requirement; a candidate is not a `CaseStudy` until an admin explicitly creates one via `useCreateCaseStudy`, and this test exists specifically to catch a future change that accidentally wires discovery results into the cache |

---

### 9.2 Test Case Detail — CaseStudyTable.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders one row per case study, with name/isPublished/evidenceTier/createdAt | `vi.mock('@/hooks/useAdminCaseStudies')` → `{data: {items: [c1, c2], page: 1, limit: 20, total: 2}}` | render `<CaseStudyTable onEdit={vi.fn()} />` | 2 rows rendered, each showing `name`, an `isPublished` badge, `evidenceTier`, and `createdAt` |
| Filter control drives the hook's isPublished argument, not client-side filtering | mocked hook; spy on call args | render, select the "Published" filter option | `useAdminCaseStudies` re-called (or its args tracked) with `isPublished: true`; select "Draft" | called with `isPublished: false`; select "All" | called with `isPublished: undefined` — confirms Doc 9-7's requirement that the table's filter maps to the query parameter, not a `.filter()` over an already-fetched list |
| Edit action calls onEdit with the row's case study | mocked hook → one item; `onEdit: vi.fn()` prop | click the row's "Edit" action | `onEdit` called with that row's case study object |
| Delete requires an explicit confirm step before mutating | mocked `useDeleteCaseStudy` → `{mutate: vi.fn()}` | click the row's "Delete" action, without confirming | `mutate` NOT called — a confirm dialog/step is shown first, per Doc 9-7's explicit "not a single accidental click" requirement |
| Confirming delete calls the mutation with the row's id | mocked `useDeleteCaseStudy` → `{mutate: vi.fn()}` | click "Delete", then confirm | `mutate` called with that row's `id` |
| Empty state reflects the active filter, not a generic "no case studies" message | mocked hook → `{data: {items: [], page: 1, limit: 20, total: 0}}` with the "Draft" filter selected | render | an empty-state message specific to the draft filter is shown (e.g. distinguishable text from the "no case studies at all" case) — per Doc 9-7's explicit callout that these two empty messages must read differently |
| Loading state | mocked hook → `{isPending: true}` | render | loading/skeleton state shown, no rows |

---

### 9.2 Test Case Detail — CaseStudyForm.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders all required and optional fields | mocked `useCreateCaseStudy`/`useUpdateCaseStudy` idle | render `<CaseStudyForm mode="create" />` | fields for `name`, `latitude`, `longitude`, `evidenceDescription`, `scoreRiseDate`, `confirmedDate` (required) and `gridCellId`, `evidenceUrl`, `evidenceTier`, `beforeImageUrl`, `afterImageUrl`, `isPublished` (optional) are all present |
| Client-side validation blocks submit when a required field is missing | mocked hook `mutate` | submit with `name` empty | `mutate` NOT called; a field-level error is shown for `name` |
| Create mode calls useCreateCaseStudy on submit | mocked `useCreateCaseStudy` → `{mutate: vi.fn()}`; mode `create` | fill all required fields, submit | `mutate` called with the form values; `useUpdateCaseStudy`'s mutate NOT called |
| Edit mode calls useUpdateCaseStudy with the given caseStudyId | mocked `useUpdateCaseStudy` → `{mutate: vi.fn()}`; `mode="edit"`, `caseStudyId={id}` | change one field, submit | `mutate` called with `{id, input: {...changed field...}}`; `useCreateCaseStudy`'s mutate NOT called |
| initialValues pre-fill the form in edit mode | props: `mode="edit"`, `initialValues={{name: 'Bole Rd Expansion', ...}}` | render | the `name` field's value is `'Bole Rd Expansion'` (and other provided fields are pre-filled accordingly) |
| initialValues from a discovery candidate pre-fill without auto-submitting | props: `mode="create"`, `initialValues` derived from a `DiscoveryCandidate` (`evidenceDescription`/`evidenceUrl`/`evidenceTier` mapped, `scoreRiseDate` from `mentionedDate`) | render | fields show the candidate-derived values; `mutate` NOT called on render — pre-filling is not submitting, per Doc 9-7 |
| Date-order-warning response renders a visible inline banner, not a toast | mocked `useCreateCaseStudy` → `mutate` resolves with a `message` containing the date-order warning text | submit, await success | an inline warning banner is rendered **inside the form**, still visible after the resolve — not dispatched as a toast/global notification, per frontend spec 6.4's explicit requirement |
| Normal success (no warning) does not render the warning banner | mocked `useCreateCaseStudy` → `mutate` resolves with a plain success message | submit, await success | no warning banner rendered; `onSuccess` prop called (if provided) |
| Same warning-banner behavior applies to edit-mode updates | mocked `useUpdateCaseStudy` → `mutate` resolves with the date-order warning message | submit in `mode="edit"`, await | inline warning banner rendered — confirms the shared response-inspection code path covers both mutations, per Doc 9-7, rather than only being implemented for create |
| 404 grid-cell-not-found attaches to the gridCellId field specifically | mocked `useCreateCaseStudy` → `mutate` rejects with `{response: {status: 404, data: {message: 'Grid cell not found'}}}` | submit with a `gridCellId` filled in, await | a field-level error is shown on `gridCellId`; no generic form-wide error banner is used for this case — per Doc 9-7's "single-field problem" framing |
| isSubmitting disables the submit button | mocked hook state `isPending: true` | render | submit button disabled |

---

### 9.2 Test Case Detail — DiscoveryPanel.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders an areaFocus input and submit control | mocked `useDiscoverCandidates` idle | render `<DiscoveryPanel onUseCandidate={vi.fn()} />` | an `areaFocus` text input and a submit control are present |
| Submitting calls the discovery mutation with the entered areaFocus | mocked `useDiscoverCandidates` → `{mutate: vi.fn()}` | type an area, submit | `mutate` called with the entered `areaFocus` string |
| Renders each candidate's summary, sourceUrl, suggestedEvidenceTier, and mentionedDate | mocked hook → success with `{candidates: [c1, c2]}` | render | for each candidate: summary text, an outbound link using `sourceUrl`, `suggestedEvidenceTier`, and `mentionedDate` are all shown |
| "Use this" calls onUseCandidate with the full candidate, and never mutates anything itself | mocked hook → success with one candidate; `onUseCandidate: vi.fn()`; also spy on `useCreateCaseStudy`/`useUpdateCaseStudy`'s `mutate` if rendered in the same tree, or otherwise assert no such mutation function is invoked | click that candidate's "Use this" button | `onUseCandidate` called with that exact candidate object; no create/update/publish mutation is called as a side effect — per Doc 9-7's explicit test requirement guarding UC-15's postcondition that a candidate can only become a `CaseStudy` by passing through `CaseStudyForm` |
| Empty candidates renders an inline no-results message, not an error | mocked hook → success with `{candidates: []}` | render after submit | "No relevant candidates found for this area" (or equivalent) shown inline; no error styling |
| Discovery-unavailable error renders distinctly from the empty-results message | mocked hook → `{isError: true, error: {response: {status: 400, data: {message: 'Discovery search is temporarily unavailable'}}}}` | render after submit | an error message is shown, visually/textually distinct from the empty-results message above — a test asserting only "some message appears" for both would not catch these being collapsed into one generic state |
| Loading state while a search is in flight | mocked hook → `{isPending: true}` | render | a loading indicator shown, no stale candidate list left over from a previous search |

---

### 9.3 Test Case Detail — CaseStudyCurationPage.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders CaseStudyTable and DiscoveryPanel | mock child components `CaseStudyTable`, `CaseStudyForm`, `DiscoveryPanel` | render `<CaseStudyCurationPage />` | both `CaseStudyTable` and `DiscoveryPanel` rendered; `CaseStudyForm` not rendered (form closed by default) |
| CaseStudyTable's onEdit opens the form in edit mode with complete initialValues | mocked `CaseStudyTable` exposing a way to trigger its `onEdit` prop with a case study object | trigger `onEdit(caseStudy)` | `CaseStudyForm` is rendered with `mode="edit"`, the matching `caseStudyId`, and `initialValues` populated — including fields not necessarily present on `CaseStudyTable`'s summary row shape (e.g. coordinates/evidence text), confirming the page — not the table — is responsible for supplying complete values, per Doc 9-7 |
| DiscoveryPanel's onUseCandidate opens the form in create mode pre-filled from the candidate | mocked `DiscoveryPanel` exposing a way to trigger its `onUseCandidate` prop with a candidate | trigger `onUseCandidate(candidate)` | `CaseStudyForm` rendered with `mode="create"` and `initialValues` derived from the candidate's `summary`/`evidenceUrl`/`suggestedEvidenceTier`/`mentionedDate` |
| Form's onSuccess closes the form | render with the form open (via either trigger above); mocked `CaseStudyForm` exposing a way to trigger its `onSuccess` prop | trigger `onSuccess` | `CaseStudyForm` no longer rendered |
| Page itself does not invalidate or refetch the table directly | mock `queryClient.invalidateQueries` at the page level (or spy for any direct call originating from the page, distinct from calls made inside the mocked hooks) | open the form, trigger `onSuccess` | no invalidate/refetch call originates from `CaseStudyCurationPage` itself — refreshing `CaseStudyTable` is left entirely to the hooks' own `onSuccess` cache invalidation (per Doc 9-7), not duplicated at the page level |

---

### 9.4 Coverage Honesty Check (per PR Steward, at review time)

- [ ] `useCreateCaseStudy` and `useUpdateCaseStudy`'s date-order-warning tests assert `isSuccess`/no thrown error, and are written as two separate test cases (create and update) rather than one covering "basically the same as the other" — the update path is a distinct regression risk per Doc 9-7
- [ ] `useDiscoverCandidates`'s no-cache-write regression guard is a real assertion against a mocked `invalidateQueries` (or equivalent spy), present in the test file — not merely described in a comment
- [ ] `DiscoveryPanel`'s "Use this never mutates" test explicitly spies on or mocks the create/update mutation functions and asserts they were not called — not inferred only from `onUseCandidate` having been called, which would not catch an implementation that (incorrectly) does both
- [ ] `CaseStudyTable`'s filter test asserts the hook was called with the correct `isPublished` argument for all three states (`true`/`false`/`undefined`), not just one, since "All" defaulting correctly back to `undefined` is as easy to regress as the two boolean states
- [ ] `CaseStudyForm`'s warning-banner test confirms the banner persists in the rendered output after success (an inline element), not merely that some success callback fired — distinguishing it from a toast requires actually checking what's in the DOM
- [ ] `useUpdateCaseStudy`'s "doubles as publish" test checks the actual request body sent (`isPublished: true`), confirming there is no separate, undocumented publish code path a future change could introduce inconsistently
- [ ] `CaseStudyCurationPage`'s "page does not invalidate directly" check is a genuine assertion isolating this responsibility to the hooks, consistent with the same pattern already established for `DashboardLayout`'s logout button (9-1) and `AdminLoginPage` (9-5)

---

### 9.5 Out of Scope for Automated Testing (and why)

- **Real AI-assisted discovery quality** — component/hook tests here mock `useDiscoverCandidates` entirely and can only verify the empty/error/success branching and the no-auto-publish guard; whether real discovered candidates are relevant and non-fabricated is a manual review concern, already covered as out-of-scope for the underlying `aiClient.ts` behavior in the backend test doc (9-7 backend).
- **Evidence tier / evidence URL content accuracy** — same as 9-3's identical note: whether a submitted `evidenceUrl` genuinely supports the claimed case study is an editorial/admin-curation judgment call (UC-14), not something these tests can or should verify; the frontend only collects and displays what the admin enters.
- **Real map-based grid-cell picking UI, if `gridCellId` is selected via a map widget rather than a raw text/id field** — this file's `CaseStudyForm` tests treat `gridCellId` as a plain form field; if the actual implementation adds a map-picker sub-component for it, that picker's own visual/interaction behavior belongs in a dedicated test file, not folded into this one.

---

**All frontend test documentation complete.**
