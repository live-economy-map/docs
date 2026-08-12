## Project: Shadow Economy Map — Frontend Specification
**Feature:** Case Studies & Validation
**Conventions:** see `0-frontend-conventions.md` — public route, no auth guard, `PublicLayout`.
**API reference:** `02-case-studies-api.md`

---

### 2.1 Routes
```
/case-studies                → PublicLayout → CaseStudiesListPage
/case-studies/:caseStudyId   → PublicLayout → CaseStudyDetailPage (also used as a modal/panel when opened from the map)
```

### 2.2 Types (src/types/index.ts)
```typescript
export interface CaseStudySummary {
  id: string;
  name: string;
  latitude: number;
  longitude: number;
  scoreRiseDate: string;
  confirmedDate: string;
  evidenceTier: 'OFFICIAL' | 'MARKET_REPORT' | 'INFRASTRUCTURE' | 'LOCAL_NEWS';
}

export interface CaseStudyDetail extends CaseStudySummary {
  evidenceDescription: string;
  evidenceUrl: string | null;
  beforeImageUrl: string | null;
  afterImageUrl: string | null;
}
```

### 2.3 Hooks (src/hooks/useCaseStudies.ts)
```typescript
export function useCaseStudies(page = 1, limit = 20) {
  return useQuery({
    queryKey: [QUERY_KEYS.CASE_STUDIES, page, limit],
    queryFn: () => api.get<{ items: CaseStudySummary[]; page: number; limit: number; total: number }>(
      '/case-studies', { params: { page, limit } }
    ).then((r) => r.data),
  });
}

export function useCaseStudyDetail(caseStudyId: string | null) {
  return useQuery({
    queryKey: [QUERY_KEYS.CASE_STUDY_DETAIL, caseStudyId],
    queryFn: () => api.get<CaseStudyDetail>(`/case-studies/${caseStudyId}`).then((r) => r.data),
    enabled: !!caseStudyId,
  });
}
```

### 2.4 Components (src/components/case-studies/)
| Component | Responsibility |
|---|---|
| `CaseStudyMarker.tsx` | Map pin, used inside `GrowthMapCanvas` (Growth Map feature) — clicking opens `CaseStudyDetailPanel` |
| `CaseStudyDetailPanel.tsx` | Renders `useCaseStudyDetail` — shows evidence, dates, and `BeforeAfterSlider` only if both image URLs are non-null; otherwise shows evidence/dates only, per API spec 2.2 |
| `BeforeAfterSlider.tsx` | Drag-reveal comparison widget for `beforeImageUrl`/`afterImageUrl` |
| `CaseStudyListRow.tsx` | Row in `CaseStudiesListPage`, links to detail |
| `EmptyState.tsx` (from `components/common/`) | Shown when `items: []` — no published case studies yet |

### 2.5 Cross-Feature Note
`CaseStudyMarker` is rendered *inside* the Growth Map feature's canvas (it's data on the same map), but its data/hooks live here, owned by this feature — the map component imports `useCaseStudies` from this file rather than duplicating the type/hook. This is the one clear cross-feature dependency in V1; call it out explicitly in any task/ticket splitting so the Growth Map owner and Case Studies owner coordinate on this one integration point.

---

**Next:** proceed to → [03. Site Content & Onboarding Frontend]
