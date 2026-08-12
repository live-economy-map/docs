## Project: Shadow Economy Map — Frontend Specification
**Feature:** Site Content & Onboarding
**Conventions:** see `0-frontend-conventions.md` — public route, no auth guard, `PublicLayout`.
**API reference:** `03-site-content-api.md`

---

### 3.1 Routes
```
/               → PublicLayout → LandingPage
/methodology    → PublicLayout → MethodologyPage
```
Onboarding overlay (map legend) has no route — it's static copy rendered conditionally inside `GrowthMapPage` (Growth Map feature), not served by this feature's API (per API spec 3.1's note).

### 3.2 Types
```typescript
export interface LandingContent {
  tagline: string;
  intro: string;
  highlightStats: { publishedCaseStudyCount: number; lastDataRefresh: string };
}

export interface MethodologyContent {
  scoreExplanation: string;
  dataSources: { key: string; name: string; description: string }[];
  validationApproach: string;
  limitations: string[];
}
```

### 3.3 Hooks (src/hooks/useContent.ts)
```typescript
export function useLandingContent() {
  return useQuery({
    queryKey: [QUERY_KEYS.CONTENT_LANDING],
    queryFn: () => api.get<LandingContent>('/content/landing').then((r) => r.data),
    staleTime: 5 * 60 * 1000,
  });
}

export function useMethodologyContent() {
  return useQuery({
    queryKey: [QUERY_KEYS.CONTENT_METHODOLOGY],
    queryFn: () => api.get<MethodologyContent>('/content/methodology').then((r) => r.data),
    staleTime: 5 * 60 * 1000,
  });
}
```

### 3.4 Components (src/components/content/)
| Component | Responsibility |
|---|---|
| `HighlightStats.tsx` | Small stat row on `LandingPage` |
| `DataSourceCard.tsx` | One card per `dataSources` entry on `MethodologyPage` |
| `LimitationsList.tsx` | Renders `limitations[]` plainly — never buried/collapsed by default, per transparency-first positioning |
| `OnboardingOverlay.tsx` | Static (no API call), dismissible, session-only — lives here structurally but is rendered from `GrowthMapPage` |

---

**Next:** proceed to → [04. Admin Access Frontend]
