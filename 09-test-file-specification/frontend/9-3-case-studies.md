## Project: Shadow Economy Map — Frontend Test Documentation: Case Studies & Validation
**Links back to:** [8-3. Frontend Function-Level Spec: Case Studies & Validation], [8-1. Frontend Test Documentation: Shared/Config], [9-2. Frontend Test Documentation: Growth Map & Exploration]

Per the Frontend Testing Guide: hooks mock `@/lib/axios`; pages/components mock this feature's own hooks (`@/hooks/useCaseStudies`), never axios. This feature owns the one clear cross-feature dependency in V1 (frontend conventions 2.5): `CaseStudyMarker` is *rendered inside* `GrowthMapCanvas`, but its data/hooks live and are tested here — 9-2's `GrowthMapCanvas` tests treat any marker as an opaque child and do not duplicate this file's coverage.

---

### 9.1 Test File Map

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| `src/hooks/useCaseStudies.ts` | `tests/hooks/useCaseStudies.test.ts` | Unit (`renderHook`, mocked `@/lib/axios`) | ☐ |
| `src/components/case-studies/CaseStudyMarker.tsx` | `tests/components/case-studies/CaseStudyMarker.test.tsx` | Component (RTL, presentational) | ☐ |
| `src/components/case-studies/CaseStudyDetailPanel.tsx` | `tests/components/case-studies/CaseStudyDetailPanel.test.tsx` | Component (RTL, mocked `useCaseStudyDetail`) | ☐ |
| `src/components/case-studies/BeforeAfterSlider.tsx` | `tests/components/case-studies/BeforeAfterSlider.test.tsx` | Component (RTL, presentational) | ☐ |
| `src/components/case-studies/CaseStudyListRow.tsx` | `tests/components/case-studies/CaseStudyListRow.test.tsx` | Component (RTL, presentational) | ☐ |
| `src/pages/CaseStudiesListPage.tsx` | `tests/pages/CaseStudiesListPage.test.tsx` | Component (RTL, mocked `useCaseStudies`) | ☐ |
| `src/pages/CaseStudyDetailPage.tsx` | `tests/pages/CaseStudyDetailPage.test.tsx` | Component (RTL, mocked `useCaseStudyDetail`) | ☐ |

`components/common/EmptyState.tsx` is exercised here (empty case-study list) but is documented and owned in whichever feature first introduces it — see that module's own file map for its dedicated test.

---

### 9.2 Test Case Detail — useCaseStudies.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| useCaseStudies calls the correct endpoint with pagination | mocked `api.get` resolves `{items, page, limit, total}` | `renderHook(() => useCaseStudies(1, 20))` | `api.get` called with `('/case-studies', {params: {page: 1, limit: 20}})` |
| Defaults to page 1, limit 20 when called with no args | mocked `api.get` resolves | `renderHook(() => useCaseStudies())` | `api.get` called with `{params: {page: 1, limit: 20}}` |
| Empty published list resolves without error | mocked `api.get` resolves `{items: [], page: 1, limit: 20, total: 0}` | render, await | `result.current.data.items` is `[]`; `isError` stays false |
| useCaseStudyDetail is disabled when caseStudyId is null | mocked `api.get` | `renderHook(() => useCaseStudyDetail(null))` | `api.get` NOT called |
| useCaseStudyDetail fetches once an id is given | mocked `api.get` resolves a `CaseStudyDetail` | `renderHook(() => useCaseStudyDetail(id))` | `api.get` called with `('/case-studies/' + id)` |
| 404 (unpublished or nonexistent) surfaces as a single error state | mocked `api.get` rejects with a 404 AxiosError | render, await | `result.current.isError === true` — hook exposes no field distinguishing "doesn't exist" from "not published," matching the backend's identical-404 contract (API spec 2.2) |

---

### 9.2 Test Case Detail — CaseStudyMarker.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders a marker for a given case study | props: `{id, name, latitude, longitude, evidenceTier}` (a `CaseStudySummary`) | render | a marker element is present, positioned/labeled per the given coordinates/name |
| Clicking the marker opens the detail panel | `onSelect: vi.fn()` prop | `userEvent.click` the marker | `onSelect` called with the marker's `id` |
| Distinct rendering per evidenceTier | render four markers, one per tier (`OFFICIAL`/`MARKET_REPORT`/`INFRASTRUCTURE`/`LOCAL_NEWS`) | render | each tier renders with a distinguishable visual treatment (icon/class), confirming the tier is exposed visually, not just stored data |

---

### 9.2 Test Case Detail — CaseStudyDetailPanel.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Calls useCaseStudyDetail with the given id | `vi.mock('@/hooks/useCaseStudies')`; mocked hook | render `<CaseStudyDetailPanel caseStudyId={id} />` | `useCaseStudyDetail` called with `id` |
| Renders evidence description, evidence link, and dates on success | mocked hook → success, full detail with non-null `evidenceUrl` | render | evidence text, a link to `evidenceUrl`, `scoreRiseDate`, and `confirmedDate` are all shown |
| Renders BeforeAfterSlider only when both image URLs are present | mocked hook → success with `beforeImageUrl` and `afterImageUrl` both non-null | render | `BeforeAfterSlider` present |
| Omits BeforeAfterSlider when either image URL is null | mocked hook → success with `beforeImageUrl: null` (or `afterImageUrl: null`) | render | `BeforeAfterSlider` NOT present; evidence/dates still shown — per API spec 2.2, imagery is a "Should," not guaranteed per case study |
| evidenceUrl null renders evidence text without a broken link | mocked hook → success with `evidenceUrl: null` | render | evidence description text shown; no link element rendered (not a link pointing at `null`/`undefined`) |
| Loading state | mocked hook → `{isPending: true}` | render | loading indicator shown, no evidence content yet |
| Error/not-found state | mocked hook → `{isError: true}` | render | a not-found/error message shown, not a blank panel |

---

### 9.2 Test Case Detail — BeforeAfterSlider.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders both images | props: `beforeImageUrl`, `afterImageUrl` (both non-null strings) | render | both images present in the DOM (e.g. via `alt`/`src`) |
| Drag interaction updates the reveal position | — | `userEvent` drag/interact with the slider handle | the reveal boundary (assert via a style/attribute reflecting position, e.g. a clip-path or width percentage) changes from its initial value |

---

### 9.2 Test Case Detail — CaseStudyListRow.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders name, dates, and evidence tier | props: a `CaseStudySummary` | render | name, `scoreRiseDate`, `confirmedDate`, and evidence tier text/badge all visible |
| Links to the case study's detail route | props include `id` | render | the row's link `href`/`to` equals `/case-studies/:id` matching the given id |

---

### 9.3 Test Case Detail — CaseStudiesListPage.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders one CaseStudyListRow per published case study | `vi.mock('@/hooks/useCaseStudies')` → `{data: {items: [c1, c2, c3], page:1, limit:20, total:3}}` | render `<CaseStudiesListPage />` | 3 rows rendered |
| Loading state shows a skeleton/placeholder | mocked hook → `{isPending: true}` | render | skeleton/placeholder shown, no rows |
| Zero published case studies shows EmptyState | mocked hook → `{data: {items: [], total: 0}}` | render | `EmptyState` rendered — per UC-08's alternate flow, this is a client-side empty-state message, never treated as an error even though `total: 0` |
| Error state | mocked hook → `{isError: true}` | render | an error message shown, distinct from the empty-state message above (zero-results and fetch-failure must not look identical to the visitor) |

---

### 9.3 Test Case Detail — CaseStudyDetailPage.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Fetches by the route's caseStudyId param | mock `useParams` → `{caseStudyId}`; `vi.mock('@/hooks/useCaseStudies')` | render `<CaseStudyDetailPage />` | `useCaseStudyDetail` called with the route param's id |
| Renders full evidence detail on success | mocked hook → success, full detail | render | same content coverage as `CaseStudyDetailPanel` (this page composes that component; assert the panel is present with the right id passed through, not re-testing every internal field again) |
| 404 renders a generic not-found message | mocked hook → `{isError: true}` (an unpublished-or-nonexistent case study, indistinguishable at this layer per the hook's own contract) | render | a generic not-found message and a link back to `/case-studies` are shown; no branch exists in this component distinguishing the two underlying causes, since the backend never does either |

---

### 9.4 Coverage Honesty Check (per PR Steward, at review time)

- [ ] `CaseStudyDetailPanel`'s "omits BeforeAfterSlider" case is tested with only ONE of the two image URLs null (not both), confirming the guard is genuinely "both must be non-null," not "at least one is non-null"
- [ ] The zero-published-case-studies test (`CaseStudiesListPage`) and the fetch-error test are asserted with visibly different UI outcomes — a test that only checks "some empty-ish message" for both wouldn't catch a regression where a real error silently renders as if there were simply nothing to show
- [ ] `CaseStudyDetailPage`'s generic-404 test confirms there is no conditional branch rendering different text for "doesn't exist" vs. "not published" — since the hook/backend never distinguishes them, a component that tried to guess would be inventing information it doesn't have
- [ ] `CaseStudyMarker`'s per-tier rendering test checks all four `EvidenceTier` values, not just one, since a missing/default case for an unhandled tier value is a realistic regression as new tiers are never added but could be reordered/renamed
- [ ] `useCaseStudies`'s default-args test (`useCaseStudies()`) is a real, separate test case from the explicit-args test — not assumed to be covered "because the explicit-args case obviously implies the defaults work"

---

### 9.5 Out of Scope for Automated Testing (and why)

- **Real map-pin placement and clustering behavior for CaseStudyMarker inside the actual map library** — covered structurally here (marker renders, click fires a callback); real geographic placement accuracy is a manual/visual QA concern shared with 9-2's map-rendering out-of-scope note.
- **Real before/after satellite imagery quality or load performance** — `BeforeAfterSlider` tests assert the drag-reveal mechanic and that both images are present in the DOM; actual image content/quality is a data-curation concern (per UC-14), not a frontend test concern.
- **Evidence tier/evidence URL content accuracy** — same as the backend test doc's own equivalent note: whether a given `evidenceUrl` genuinely supports the claimed case study is an editorial/admin-curation concern (see 9-7), not something the public-facing display components can or should verify.

---

**Next:** proceed to → [9-4. Frontend Test Documentation: Site Content & Onboarding]
