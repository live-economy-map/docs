## Project: Shadow Economy Map — Frontend Specification
**Feature:** Growth Map & Exploration
**Conventions:** see `0-frontend-conventions.md` — public route, no auth guard, `PublicLayout`.
**API reference:** `01-growth-map-api.md`

---

### 1.1 Route

```
/map → PublicLayout → GrowthMapPage (default export, src/pages/map/GrowthMapPage.tsx)
```

### 1.2 Types (added to src/types/index.ts)

```typescript
export interface GrowthCell {
  cellId: string;
  cellRow: number;
  cellCol: number;
  boundaryGeoJson: GeoJSON.Polygon;
  compositeScore: number;
  isComplete: boolean;
}

export interface CellDetail {
  cellId: string;
  period: string;
  areaLabel: string | null;
  compositeScore: number;
  isComplete: boolean;
  trend: 'up' | 'down' | 'flat';
  sparkline: { period: string; compositeScore: number }[];
  signals: { source: 'VIIRS' | 'GHSL' | 'RWI'; rawValue: number; normalizedValue: number }[];
  aiSummary: string | null;
  lastUpdated: string;
}

export interface RawLayerCell {
  cellId: string;
  normalizedValue: number;
}

export interface MapSearchResult {
  parsedFilters: { areaLabel?: string; period?: string; signalFocus?: string } | null;
  cells: { cellId: string; compositeScore: number }[];
}
```

### 1.3 Hooks (src/hooks/useMap.ts)

```typescript
export function useMapCells(period?: string) {
  return useQuery({
    queryKey: [QUERY_KEYS.MAP_CELLS, period],
    queryFn: () => api.get<{ period: string; periodSubstituted: boolean; cells: GrowthCell[] }>(
      '/map/cells', { params: { period } }
    ).then((r) => r.data),
  });
}

export function useCellDetail(cellId: string | null, period?: string) {
  return useQuery({
    queryKey: [QUERY_KEYS.CELL_DETAIL, cellId, period],
    queryFn: () => api.get<CellDetail>(`/map/cells/${cellId}`, { params: { period } }).then((r) => r.data),
    enabled: !!cellId, // only fetch once a cell is actually selected
  });
}

export function useRawLayer(sourceKey: 'VIIRS' | 'GHSL' | 'RWI' | null, period?: string) {
  return useQuery({
    queryKey: [QUERY_KEYS.MAP_LAYER, sourceKey, period],
    queryFn: () => api.get<{ cells: RawLayerCell[] }>(`/map/layers/${sourceKey}`, { params: { period } }).then((r) => r.data),
    enabled: !!sourceKey, // only fetch when a raw layer is actively toggled on, not the composite default
  });
}

export function useMapSearch() {
  return useMutation({
    mutationFn: (query: string) => api.post<MapSearchResult>('/map/search', { query }).then((r) => r.data),
  });
}
```

### 1.4 Local (non-global) State

Per conventions 0.3, these are `useState`/URL search params on `GrowthMapPage`, **not** Zustand — they're view state, not app-wide state:
- `selectedPeriod: string | undefined` — drives `useMapCells`/`useCellDetail`, reflected in the URL (`?period=...`) so a shared link preserves the view
- `selectedCellId: string | null` — drives `useCellDetail`, opens/closes the detail panel
- `activeRawLayer: 'VIIRS' | 'GHSL' | 'RWI' | null` — `null` means composite view (default)

### 1.5 Components (src/components/map/)

| Component | Responsibility |
|---|---|
| `GrowthMapCanvas.tsx` | Renders the map (Mapbox GL/Leaflet per conventions), shades cells from `useMapCells`/`useRawLayer`, handles click → sets `selectedCellId` |
| `TimeSlider.tsx` | Controls `selectedPeriod`; shows a note when `periodSubstituted: true` is returned |
| `CellDetailPanel.tsx` | Renders `useCellDetail`'s data — numeric fields render immediately; `aiSummary` shows a skeleton/loading state until present, and a fallback line if `null` (per UC-03's graceful-degradation requirement) |
| `LayerToggle.tsx` | Switches `activeRawLayer`; only ever offers `VIIRS \| GHSL \| RWI`, never `GDELT` (per API spec 1.2's explicit exclusion) |
| `MapSearchBar.tsx` | Wraps `useMapSearch`; on a `parsedFilters: null` response, shows the returned `message` inline and leaves the current map view untouched, per UC-10's alternate flow |

### 1.6 Page Composition (src/pages/map/GrowthMapPage.tsx)

Thin, per template convention — composes `GrowthMapCanvas`, `TimeSlider`, `LayerToggle`, `MapSearchBar`, and conditionally `CellDetailPanel` (only when `selectedCellId` is set). Owns the local state listed in 1.4 and passes it down; no business logic lives in the page itself.

---

**Next:** proceed to → [02. Case Studies & Validation Frontend]
