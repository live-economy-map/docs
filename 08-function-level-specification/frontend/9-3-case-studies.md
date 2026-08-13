## Project: Shadow Economy Map — Frontend Function-Level Spec: Case Studies & Validation
Files covered: `src/pages/CaseStudiesListPage.tsx` · `src/pages/CaseStudyDetailPage.tsx` · `src/components/case-studies/*` · `src/hooks/useCaseStudies.ts`

---

### Shared Pattern: Simple Query Hook (see 9-1 for the pattern definition)

| Hook | Endpoint | Query key | Params |
|---|---|---|---|
| useCaseStudies | GET /case-studies | [QUERY_KEYS.CASE_STUDIES, page, limit] | page (default 1), limit (default 20) |
| useCaseStudyDetail | GET /case-studies/:caseStudyId | [QUERY_KEYS.CASE_STUDY_DETAIL, caseStudyId] | caseStudyId — `enabled: !!caseStudyId` |

Both are read-only, unauthenticated, and return an empty/`404` state that is rendered, never treated as a page-breaking error (per API spec 2.2 and frontend spec 2.5).

---

### src/components/case-studies/CaseStudyMarker.tsx (new — light block)

| Field | Detail |
|---|---|
| Props | `{ caseStudy: CaseStudySummary; isSelected: boolean; onClick: (caseStudyId: string) => void }` |
| Local state | None |
| Behavior | Renders a pin at `(latitude, longitude)` inside `GrowthMapCanvas` (Growth Map feature owns the canvas; this feature owns the marker per frontend spec 2.5's cross-feature note); visually distinct marker style from grid-cell shading so a case study is never mistaken for a scored cell, per NFR "Usability — distinguishes detected signal from confirmed case study" |
| Edge cases | None — data always present, this component never fetches |
| Test file | `tests/components/case-studies/CaseStudyMarker.test.tsx` |

### src/components/case-studies/CaseStudyDetailPanel.tsx (new — full block)

| Field | Detail |
|---|---|
| Props | `{ caseStudyId: string; onClose?: () => void }` |
| Local state | None — data comes from `useCaseStudyDetail(caseStudyId)` internally |
| Behavior | Renders `evidenceDescription`, `evidenceUrl` (as an outbound link, if present), `evidenceTier` badge, `scoreRiseDate`, and `confirmedDate` once the query resolves; renders `BeforeAfterSlider` only when both `beforeImageUrl` and `afterImageUrl` are non-null, otherwise omits the comparison section entirely rather than rendering a broken/placeholder slider (per UC-05's alternate flow and frontend spec 2.4) |
| Edge cases | `evidenceUrl: null` → render the description text only, no dead link. Used both as the map's click-through panel (UC-05) and as the body of `CaseStudyDetailPage` (UC-08) — `onClose` is only passed/rendered in the map context; the standalone page context omits it |
| Test file | `tests/components/case-studies/CaseStudyDetailPanel.test.tsx` |

### src/components/case-studies/BeforeAfterSlider.tsx (new — light block)

| Field | Detail |
|---|---|
| Props | `{ beforeImageUrl: string; afterImageUrl: string; beforeLabel?: string; afterLabel?: string }` |
| Local state | `sliderPosition: number` (0–100, drag-controlled) |
| Behavior | Drag-reveal comparison — `afterImageUrl` clipped to `sliderPosition`% over `beforeImageUrl`; keyboard-operable (arrow keys move `sliderPosition`) per NFR "Accessibility — usable via touch and keyboard" |
| Edge cases | Image load failure on either URL — falls back to a plain "before/after imagery unavailable" note rather than a broken-image icon; this component is only ever mounted when both URLs are non-null (caller's responsibility, see `CaseStudyDetailPanel` above), so it does not itself branch on nullability |
| Test file | `tests/components/case-studies/BeforeAfterSlider.test.tsx` |

### src/components/case-studies/CaseStudyListRow.tsx (new — light block)

| Field | Detail |
|---|---|
| Props | `{ caseStudy: CaseStudySummary; onClick: (caseStudyId: string) => void }` |
| Local state | None |
| Behavior | Renders `name`, `evidenceTier` badge, `scoreRiseDate`/`confirmedDate` as a compact row; `onClick` navigates to `ROUTES.CASE_STUDY_DETAIL` (built with `caseStudyId`) |
| Edge cases | None |
| Test file | `tests/components/case-studies/CaseStudyListRow.test.tsx` |

---

### src/pages/CaseStudiesListPage.tsx (new)

| Field | Detail |
|---|---|
| Purpose | Standalone browsable list of all published case studies, independent of the map (UC-08). |
| Route guard + layout | Public, `PublicLayout` |
| Local state | `page: number` (synced to `?page=` URL param) |
| Behavior | 1. `useCaseStudies(page)`. 2. Renders one `CaseStudyListRow` per `items` entry, paginated per API spec 2.1's `page`/`limit` convention. 3. Renders `EmptyState` (from `components/common/`) when `items: []`, per UC-08's alternate flow — not an error state. |
| Components | CaseStudyListRow, EmptyState (from `components/common/`) |

**States:** loading (skeleton rows) · error (inline + retry) · empty (`items: []` → `EmptyState`, not blocked) · success

---

### src/pages/CaseStudyDetailPage.tsx (new)

| Field | Detail |
|---|---|
| Purpose | Full evidence detail for one case study, reachable directly by URL (e.g., shared link) in addition to being opened as a panel from the map (UC-05, UC-08). |
| Route guard + layout | Public, `PublicLayout` |
| Local state | None — `caseStudyId` comes from the route param |
| Behavior | Renders `CaseStudyDetailPanel` for `caseStudyId` from `useParams()`, without `onClose` (there is nothing to close — this is the page itself, not an overlay) |
| Components | CaseStudyDetailPanel |

**States:** loading (skeleton) · error (404 → "Case study not found" page-level message, distinct from a network error — per API spec 2.2, an unpublished or nonexistent `caseStudyId` returns the same 404) · success

---

**Next:** proceed to → [9-4. Frontend: Site Content & Onboarding]
