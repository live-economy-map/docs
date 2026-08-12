## Project: Shadow Economy Map — Frontend Specification
**Feature:** Case Study Curation
**Conventions:** see `0-frontend-conventions.md` — `ProtectedRoute` + `DashboardLayout` throughout.
**API reference:** `06-case-study-curation-api.md`

---

### 6.1 Route
```
/admin/case-studies → ProtectedRoute → DashboardLayout → CaseStudyCurationPage
```

### 6.2 Types
```typescript
export interface AdminCaseStudy {
  id: string;
  name: string;
  isPublished: boolean;
  evidenceTier: string | null;
  createdAt: string;
}

export interface CaseStudyFormInput {
  name: string;
  latitude: number;
  longitude: number;
  gridCellId?: string;
  evidenceDescription: string;
  evidenceUrl?: string;
  evidenceTier?: 'OFFICIAL' | 'MARKET_REPORT' | 'INFRASTRUCTURE' | 'LOCAL_NEWS';
  scoreRiseDate: string;
  confirmedDate: string;
  beforeImageUrl?: string;
  afterImageUrl?: string;
  isPublished?: boolean;
}

export interface DiscoveryCandidate {
  summary: string;
  sourceUrl: string;
  suggestedEvidenceTier: string;
  mentionedDate: string;
}
```

### 6.3 Hooks (src/hooks/useAdminCaseStudies.ts)
```typescript
export function useAdminCaseStudies(isPublished?: boolean, page = 1) {
  return useQuery({
    queryKey: [QUERY_KEYS.ADMIN_CASE_STUDIES, isPublished, page],
    queryFn: () => api.get('/admin/case-studies', { params: { isPublished, page } }).then((r) => r.data),
  });
}

export function useCreateCaseStudy() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (input: CaseStudyFormInput) => api.post('/admin/case-studies', input),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.ADMIN_CASE_STUDIES] }),
  });
}

export function useUpdateCaseStudy() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ id, input }: { id: string; input: Partial<CaseStudyFormInput> }) =>
      api.patch(`/admin/case-studies/${id}`, input),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.ADMIN_CASE_STUDIES] }),
  });
}

export function useDeleteCaseStudy() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (id: string) => api.delete(`/admin/case-studies/${id}`),
    onSuccess: () => qc.invalidateQueries({ queryKey: [QUERY_KEYS.ADMIN_CASE_STUDIES] }),
  });
}

export function useDiscoverCandidates() {
  return useMutation({
    mutationFn: (areaFocus: string) =>
      api.post<{ candidates: DiscoveryCandidate[] }>('/admin/case-studies/discover', { areaFocus }).then((r) => r.data),
  });
}
```

### 6.4 Components (src/components/admin-case-studies/)
| Component | Responsibility |
|---|---|
| `CaseStudyTable.tsx` | List from `useAdminCaseStudies`, published/draft filter, edit/delete actions |
| `CaseStudyForm.tsx` | React Hook Form + Zod, used for both create and edit; if the create response's `message` indicates a date-order warning (`scoreRiseDate` after `confirmedDate`, per API spec 6.2), the form surfaces this as a visible inline warning banner, not just a toast |
| `DiscoveryPanel.tsx` | Wraps `useDiscoverCandidates`; each candidate has a "Use this" button that pre-fills `CaseStudyForm` with the suggested evidence — never auto-submits, per the manual-verification requirement (UC-15) |

### 6.5 Important UX Rule
Per UC-15's postcondition, `DiscoveryPanel` must never let a candidate become a published `CaseStudy` without the admin passing through `CaseStudyForm` and explicitly submitting — no "one-click publish from AI suggestion" shortcut, even though it would be technically easy to build. This is a deliberate constraint, not an oversight, and should not be "optimized away" during implementation.

---

**All frontend feature files complete.** Next: proceed to → [System Architecture]
