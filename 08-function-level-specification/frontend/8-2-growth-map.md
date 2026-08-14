## Project: Shadow Economy Map — Frontend Function-Level Spec: Growth Map & Exploration
Files covered: `src/pages/map/GrowthMapPage.tsx` · `src/components/map/*` · `src/hooks/useMap.ts`

---

### Shared Pattern: Simple Query Hook (see 9-1 for the pattern definition)

| Hook | Endpoint | Query key | Params |
|---|---|---|---|
| useMapCells | GET /map/cells | [QUERY_KEYS.MAP_CELLS, period] | period (optional) |
| useCellDetail | GET /map/cells/:cellId | [QUERY_KEYS.CELL_DETAIL, cellId, period] | cellId, period — `enabled: !!cellId` |
| useRawLayer | GET /map/layers/:sourceKey | [QUERY_KEYS.MAP_LAYER, sourceKey, period] | sourceKey, period — `enabled: !!sourceKey` |

### useMapSearch (mutation, new — full block)

| Field | Detail |
|---|---|
| Signature | `useMapSearch(): UseMutationResult<MapSearchResult, AxiosError, string>` |
| Purpose | Wraps `POST /map/search`. |
| Side effects | No query invalidation — search results are consumed directly by the caller, not cached |
| Edge cases | A `parsedFilters: null` response is a **success**, not a mutation error — `MapSearchBar` must branch on the response body's `parsedFilters` field, not on `isError` |
| Test file | `tests/hooks/useMapSearch.test.ts` |

---

### src/components/map/GrowthMapCanvas.tsx (new — full block)

| Field | Detail |
|---|---|
| Props | `{ cells: GrowthCell[] \| RawLayerCell[]; onCellClick: (cellId: string) => void; selectedCellId: string \| null }` |
| Local state | None — rendering is fully derived from props |
| Behavior | Renders the map base layer, draws each cell's `boundaryGeoJson` shaded by `compositeScore` (or `normalizedValue` for raw layers), highlights `selectedCellId` if set, click handler calls `onCellClick(cellId)` |
| Edge cases | `isComplete: false` cells (composite view only) render with a distinct visual treatment (e.g., hatched pattern) rather than a solid color, so incomplete data is never visually indistinguishable from a confident score |
| Test file | `tests/components/map/GrowthMapCanvas.test.tsx` |

### src/components/map/TimeSlider.tsx (new — full block)

| Field | Detail |
|---|---|
| Props | `{ availablePeriods: string[]; selectedPeriod: string \| undefined; onChange: (period: string) => void; periodSubstituted: boolean }` |
| Local state | None — controlled entirely by props, per the page owning `selectedPeriod` (frontend spec 1.4) |
| Behavior | Renders a draggable/steppable slider across `availablePeriods`; calls `onChange` on interaction |
| Edge cases | When `periodSubstituted: true`, renders a small inline note ("showing nearest available period") next to the slider — this must never be silently absorbed, since the visitor is looking at a different period than they selected |
| Test file | `tests/components/map/TimeSlider.test.tsx` |

### src/components/map/CellDetailPanel.tsx (new — full block)

| Field | Detail |
|---|---|
| Props | `{ cellId: string; period: string \| undefined; onClose: () => void }` |
| Local state | None — data comes from `useCellDetail(cellId, period)` internally |
| Behavior | Renders composite score, trend arrow, sparkline, area label, and the 3 raw signals immediately once the query resolves; `aiSummary` section renders its own skeleton while `data.aiSummary` is present-but-loading is not distinguishable from the rest of the payload being loaded — since the API returns the full payload in one response (API spec 1.2), the "AI still generating" case shows as a brief overall loading state, not a separately-delayed summary field, unless the backend team later splits this into two calls |
| Edge cases | `aiSummary: null` → render a plain "Summary unavailable" line, not an empty section. `trend: 'flat'` (no prior period) → render a neutral indicator, not up/down |
| Test file | `tests/components/map/CellDetailPanel.test.tsx` |

### src/components/map/LayerToggle.tsx (new — light block)

| Field | Detail |
|---|---|
| Props | `{ activeLayer: 'VIIRS' \| 'GHSL' \| 'RWI' \| null; onChange: (layer: 'VIIRS' \| 'GHSL' \| 'RWI' \| null) => void }` |
| Behavior | Renders 4 options: Composite (maps to `null`), VIIRS, GHSL, RWI — **GDELT is never an option here**, per API spec 1.2's explicit exclusion; this must not be "completed" by someone adding a 4th toggle option later without re-reading that decision |

### src/components/map/MapSearchBar.tsx (new — full block)

| Field | Detail |
|---|---|
| Props | `{ onResult: (cells: { cellId: string; compositeScore: number }[]) => void }` |
| Local state | `queryText: string` |
| Behavior | On submit, calls `useMapSearch().mutate(queryText)`; on a response with non-null `parsedFilters`, calls `onResult(response.cells)`; on `parsedFilters: null`, displays `response.message` inline near the search bar and does **not** call `onResult` (leaves the current map view untouched, per UC-10's alternate flow) |
| Edge cases | The 400 "Search is temporarily unavailable" case (AI service down) is a genuine mutation error — shown as a toast/inline error distinct from the "couldn't understand" in-band message above; these two must not be visually conflated, since one is a client explanation and the other is a service outage |
| Test file | `tests/components/map/MapSearchBar.test.tsx` |

---

### src/pages/map/GrowthMapPage.tsx (new)

| Field | Detail |
|---|---|
| Purpose | The core map experience (UC-01 through UC-04, UC-10). |
| Route guard + layout | Public, `PublicLayout` |
| Local state | `selectedPeriod: string \| undefined` (synced to `?period=` URL param), `selectedCellId: string \| null`, `activeRawLayer: 'VIIRS' \| 'GHSL' \| 'RWI' \| null` |
| Behavior | 1. `useMapCells(selectedPeriod)` or `useRawLayer(activeRawLayer, selectedPeriod)` depending on `activeRawLayer`. 2. Renders `TimeSlider`, `LayerToggle`, `MapSearchBar`, `GrowthMapCanvas` (passing whichever cell data is active). 3. Renders `CellDetailPanel` only when `selectedCellId` is set. 4. Renders `OnboardingOverlay` on first load per session (not persisted across sessions, per frontend spec 3.4). |
| Components | GrowthMapCanvas, TimeSlider, LayerToggle, MapSearchBar, CellDetailPanel, OnboardingOverlay |

**States:** loading (initial cell fetch) · error (inline + retry) · empty (`cells: []` — render the base map with no shading, not a blocked page) · success

---

**Next:** proceed to → [9-3. Frontend: Case Studies & Validation]
