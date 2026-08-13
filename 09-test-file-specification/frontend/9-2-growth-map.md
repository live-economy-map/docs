## Project: Shadow Economy Map — Frontend Test Documentation: Growth Map & Exploration
**Links back to:** [9-2. Frontend Function-Level Spec: Growth Map & Exploration], [9-1. Frontend Test Documentation: Shared/Config]

Per the Frontend Testing Guide: test-first, mirrors `src/` under `tests/`. Hooks are tested with `renderHook` + a fresh `QueryClientProvider` (via `tests/utils/renderWithProviders.tsx`), mocking `@/lib/axios` — never the network. Components/pages mock the hooks they call (`@/hooks/useMap`), never axios. `userEvent`, not `fireEvent`. This is the one feature with a documented cross-feature dependency (Case Studies' `CaseStudyMarker` renders inside `GrowthMapCanvas`) — see 9-3's cross-feature note; `GrowthMapCanvas`'s own tests here treat that marker as an opaque child and do not re-test it.

---

### 9.1 Test File Map

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| `src/hooks/useMap.ts` | `tests/hooks/useMap.test.ts` | Unit (`renderHook`, mocked `@/lib/axios`) | ☐ |
| `src/components/map/GrowthMapCanvas.tsx` | `tests/components/map/GrowthMapCanvas.test.tsx` | Component (RTL, presentational, mocked callback props) | ☐ |
| `src/components/map/TimeSlider.tsx` | `tests/components/map/TimeSlider.test.tsx` | Component (RTL, presentational, mocked callback props) | ☐ |
| `src/components/map/CellDetailPanel.tsx` | `tests/components/map/CellDetailPanel.test.tsx` | Component (RTL, mocked `useCellDetail`) | ☐ |
| `src/components/map/LayerToggle.tsx` | `tests/components/map/LayerToggle.test.tsx` | Component (RTL, presentational, mocked callback props) | ☐ |
| `src/components/map/MapSearchBar.tsx` | `tests/components/map/MapSearchBar.test.tsx` | Component (RTL, mocked `useMapSearch`) | ☐ |
| `src/pages/map/GrowthMapPage.tsx` | `tests/pages/map/GrowthMapPage.test.tsx` | Component (RTL, mocked `useMap.ts` exports) | ☐ |

---

### 9.2 Test Case Detail — useMap.test.ts

#### useMapCells / useCellDetail / useRawLayer (shared query-hook pattern)

| Case | Setup | Action | Expected result |
|---|---|---|---|
| useMapCells calls the correct endpoint with period | mocked `api.get` resolves `{period, periodSubstituted, cells}` | `renderHook(() => useMapCells('2026-06-01'))` | `api.get` called with `('/map/cells', {params: {period: '2026-06-01'}})` |
| useMapCells omits period when not given | mocked `api.get` resolves | `renderHook(() => useMapCells())` | `api.get` called with `params: {period: undefined}` — resolution of "most recent" is a backend concern, per Doc 8-2 |
| useMapCells query key includes period | inspect cache | render with two different periods | two distinct cache entries under `[QUERY_KEYS.MAP_CELLS, period]` |
| useCellDetail is disabled when cellId is null | mocked `api.get` | `renderHook(() => useCellDetail(null, undefined))` | `api.get` NOT called — `enabled: !!cellId` prevents the fetch |
| useCellDetail fetches once a cellId is provided | mocked `api.get` resolves a `CellDetail` shape | `renderHook(() => useCellDetail(cellId, period))` | `api.get` called with `('/map/cells/' + cellId, {params: {period}})` |
| useRawLayer is disabled when sourceKey is null | mocked `api.get` | `renderHook(() => useRawLayer(null, period))` | `api.get` NOT called — composite view (default) never triggers a raw-layer fetch |
| useRawLayer fetches when a source is toggled on | mocked `api.get` resolves `{cells}` | `renderHook(() => useRawLayer('VIIRS', period))` | `api.get` called with `('/map/layers/VIIRS', {params: {period}})` |
| Envelope is already unwrapped by the time the hook resolves | mocked `api.get` resolves the already-unwrapped payload shape (per 9-1's axios contract) | render, await success | `result.current.data` matches the mocked payload directly — no assertion anywhere on `statusCode`/`success`/`message` |

#### useMapSearch (mutation)

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Calls the correct endpoint | mocked `api.post` resolves `{parsedFilters: {...}, cells: [...]}` | `renderHook(() => useMapSearch())`, `.mutate('areas near Bole')` | `api.post` called with `('/map/search', {query: 'areas near Bole'})` |
| A `parsedFilters: null` response is a mutation success, not an error | mocked `api.post` resolves `{parsedFilters: null, cells: []}` | `.mutate(query)`, await | `result.current.isSuccess === true`, `result.current.isError === false` — this is a **200**-shaped "couldn't understand" response per API spec 1.2, and `useMapSearch` must not treat it as a failure |
| No cache invalidation on success | spy on `queryClient.invalidateQueries` | `.mutate(query)`, await success | `invalidateQueries` NOT called — search results are consumed directly by the caller, not cached, per Doc 9-2 |
| Genuine service-unreachable error surfaces as isError | mocked `api.post` rejects with an AxiosError (400, "Search is temporarily unavailable") | `.mutate(query)` | `result.current.isError === true` — this is the case that must stay distinguishable from the `parsedFilters: null` success case above, both at the hook layer and later at the `MapSearchBar` layer |

---

### 9.2 Test Case Detail — GrowthMapCanvas.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders cells shaded by compositeScore | props: `cells: GrowthCell[]` with varying `compositeScore` | render | one shaded shape per cell (assert count matches `cells.length`, via role/test-id per cell, not pixel color) |
| Renders raw-layer cells shaded by normalizedValue | props: `cells: RawLayerCell[]` | render | same per-cell count assertion, confirms the canvas accepts either shape per its documented prop union |
| Clicking a cell calls onCellClick with its id | props include a known `cellId`; `onCellClick: vi.fn()` | `userEvent.click` the cell element | `onCellClick` called with that exact `cellId` |
| Highlights the selectedCellId | props: `selectedCellId` set to one of the rendered cells' ids | render | that specific cell element carries the "selected" visual treatment; no other cell does |
| Incomplete cells render with a distinct (hatched) treatment | props: one cell with `isComplete: false` among several `isComplete: true` cells (composite view only) | render | the incomplete cell has a distinguishing class/attribute the complete cells do not — per Doc 9-2, incomplete data must never be visually indistinguishable from a confident score |
| Raw-layer cells never apply the incomplete-cell treatment | props: `RawLayerCell[]` (no `isComplete` field at all) | render | no cell renders the hatched/incomplete treatment — that visual state is defined only for the composite view's `isComplete` field |

---

### 9.2 Test Case Detail — TimeSlider.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders across availablePeriods | props: `availablePeriods: [3 ISO dates]`, `selectedPeriod` one of them | render | slider reflects the given range/position |
| Interaction calls onChange with the selected period | props include `onChange: vi.fn()` | `userEvent` interact with the slider control (drag/step) | `onChange` called with a period from `availablePeriods` |
| periodSubstituted renders an inline note | props: `periodSubstituted: true` | render | a note (e.g. "showing nearest available period") is visible next to the slider |
| periodSubstituted false renders no note | props: `periodSubstituted: false` | render | no substitution note present — per Doc 9-2, this must never be silently absorbed either way; both states are asserted, not just the true case |
| Component is fully controlled — no internal state drives its own position | props change `selectedPeriod` between two renders without any user interaction | re-render with new props | displayed position updates to match the new prop, confirming no local `useState` is overriding parent-controlled position |

---

### 9.2 Test Case Detail — CellDetailPanel.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Calls useCellDetail with the given cellId and period | `vi.mock('@/hooks/useMap')`; mocked `useCellDetail` | render `<CellDetailPanel cellId={id} period={period} onClose={fn} />` | `useCellDetail` called with `(id, period)` |
| Numeric fields render immediately while loading | mocked `useCellDetail` → `{isPending: true, data: undefined}` | render | a loading state is shown; per Doc 9-2 this may present as one overall skeleton (API returns the full payload in one response — see note below), not a separately-delayed AI summary field |
| Renders composite score, trend arrow, sparkline, and all 3 raw signals on success | mocked `useCellDetail` → success with full `CellDetail` data | render | composite score value, a trend indicator, a sparkline element, and 3 signal rows (VIIRS/GHSL/RWI) are all present |
| aiSummary null renders a fallback line, not an empty section | mocked `useCellDetail` → success with `aiSummary: null`, all other fields populated | render | text reading a plain "Summary unavailable"-equivalent line is shown; no empty/missing section |
| trend 'flat' renders a neutral indicator | mocked `useCellDetail` → success with `trend: 'flat'` | render | a neutral (not up/down) trend indicator is shown |
| Close button calls onClose | `onClose: vi.fn()` prop | `userEvent.click` the close control | `onClose` called |
| Error state | mocked `useCellDetail` → `{isError: true}` | render | an error message is shown, not a blank panel |

---

### 9.2 Test Case Detail — LayerToggle.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Offers exactly VIIRS, GHSL, RWI as options | render | inspect available toggle options | 3 options present, labeled/valued VIIRS/GHSL/RWI |
| Never offers GDELT as an option | render | inspect available toggle options | no GDELT option exists anywhere in the rendered control — per API spec 1.2's explicit exclusion; this must be a structural absence, not merely an unselected default |
| Selecting a source calls the change handler with that key | `onChange: vi.fn()` prop | `userEvent.click` the VIIRS option | handler called with `'VIIRS'` |
| Selecting "composite" (default) calls the change handler with null | `onChange: vi.fn()` prop; a control to return to composite view exists | `userEvent.click` the composite/default option | handler called with `null` — matches `activeRawLayer: null` meaning composite view per frontend spec 1.4 |

---

### 9.2 Test Case Detail — MapSearchBar.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders a text input and submit control | `vi.mock('@/hooks/useMap')`; mocked `useMapSearch` idle state | render | search input and submit control present |
| Submitting calls the mutation with the typed query | mocked `useMapSearch().mutate` | `userEvent.type` a query, submit | `mutate` called with the typed string |
| A confident-parse success applies filters without an inline message | mocked `useMapSearch` state simulating a resolved `{parsedFilters: {...}, cells: [...]}` | submit and resolve | no "couldn't understand" message shown; the page-level filter application is triggered (assert via a callback prop, e.g. `onResults`, being called with the cells) |
| A `parsedFilters: null` result shows the explanatory message and leaves the map view untouched | mocked `useMapSearch` state simulating a resolved `{parsedFilters: null, cells: []}` | submit and resolve | an inline message is shown; the `onResults`/filter-apply callback is NOT called — per UC-10's alternate flow, the current map view stays as-is |
| A genuine service-unreachable error shows a distinct error message | mocked `useMapSearch` state simulating `isError: true` | submit | an error message is shown, and it is visually/textually distinct from the `parsedFilters: null` inline message above — these are two different failure/non-failure paths and must not collapse into the same UI branch |
| Loading state disables the submit control | mocked `useMapSearch` state `isPending: true` | render | submit control disabled |

---

### 9.3 Test Case Detail — GrowthMapPage.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Composes canvas, slider, layer toggle, search bar | `vi.mock('@/hooks/useMap')` with idle/success states for all hooks | render `<GrowthMapPage />` | `GrowthMapCanvas`, `TimeSlider`, `LayerToggle`, `MapSearchBar` are all present |
| CellDetailPanel renders only when a cell is selected | initial render with no cell clicked | render | `CellDetailPanel` NOT present initially |
| Clicking a cell opens CellDetailPanel | mocked `useMapCells` resolves cells; click a cell in the (real, unmocked) `GrowthMapCanvas` | `userEvent.click` a cell | `CellDetailPanel` now present, receiving the clicked cell's id |
| selectedPeriod state is reflected in the URL | interact with the (real) `TimeSlider` to change period | change period | URL search params (`?period=...`) update to match — a shared link preserves the view, per frontend spec 1.4 |
| Page owns state, no business logic duplicated | — | — | (structural note, not a separate assertion) page-level tests should not need to re-verify canvas/slider internal logic already covered in their own files — this file only tests composition and state wiring |

---

### 9.4 Coverage Honesty Check (per PR Steward, at review time)

- [ ] The GDELT-absent case in `LayerToggle.test.tsx` asserts a structural absence (query for the option and expect it not found), not merely that GDELT wasn't the initially-selected value
- [ ] `MapSearchBar`'s `parsedFilters: null` case and its genuine-error case are tested as two distinct branches with two distinct expected UI outcomes, not collapsed into one generic "shows a message" assertion
- [ ] `useMapSearch`'s "success despite null parsedFilters" case is actually asserted via `isSuccess`/`isError`, not inferred from the mock simply not rejecting
- [ ] `CellDetailPanel`'s `aiSummary: null` case is tested with all other fields populated, confirming the rest of the payload still renders — not a test that only checks the summary line in isolation
- [ ] `GrowthMapCanvas`'s incomplete-cell treatment test includes both an `isComplete: true` and an `isComplete: false` cell in the same render, and asserts the two are visually distinguishable from each other — not just that the false case has "some" extra class
- [ ] `GrowthMapPage`'s click-to-open-detail-panel test exercises the real `GrowthMapCanvas` (not a mock), since this is the one integration point in this file worth not mocking away entirely

---

### 9.5 Out of Scope for Automated Testing (and why)

- **Actual Mapbox GL/Leaflet rendering, pan/zoom, and tile loading** — `GrowthMapCanvas`'s tests assert on the React-rendered cell markup and click handling, not on the underlying map library's real rendering engine; that needs manual/visual QA or a separate integration pass against the real map library.
- **Real LLM-driven natural-language parsing quality** — `useMapSearch` and `MapSearchBar` tests mock the hook/axios boundary entirely; whether the backend's actual parse for a given query is *sensible* is a backend/prompt-design concern (see the backend test doc's own equivalent out-of-scope note), not something this frontend suite can verify.
- **The 3-second/1.5-second map-load and time-slider performance NFRs** — these are real-browser timing concerns, not something a jsdom-based component test measures meaningfully.
- **Cross-feature integration with the real `CaseStudyMarker`** — `GrowthMapCanvas`'s tests treat any marker overlay as out of this file's scope; that component's own behavior is covered in 9-3, and the map/case-studies integration point itself is a shared responsibility flagged in both docs, not duplicated test coverage in either.

---

**Next:** proceed to → [9-3. Frontend Test Documentation: Case Studies & Validation]
