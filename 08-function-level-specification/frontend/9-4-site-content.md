## Project: Shadow Economy Map — Frontend Function-Level Spec: Site Content & Onboarding
Files covered: `src/pages/LandingPage.tsx` · `src/pages/MethodologyPage.tsx` · `src/components/content/*` · `src/hooks/useContent.ts`

---

### Shared Pattern: Simple Query Hook (see 9-1 for the pattern definition)

| Hook | Endpoint | Query key | Params |
|---|---|---|---|
| useLandingContent | GET /content/landing | [QUERY_KEYS.CONTENT_LANDING] | none — `staleTime: 5 * 60 * 1000` (static-ish content, per frontend spec 3.3) |
| useMethodologyContent | GET /content/methodology | [QUERY_KEYS.CONTENT_METHODOLOGY] | none — `staleTime: 5 * 60 * 1000` |

Both endpoints are unauthenticated, side-effect-free, and have no error branches beyond the generic network-failure case (per API spec 3.2 — no documented per-endpoint error conditions beyond the common statuses in API conventions 0.1).

---

### src/components/content/HighlightStats.tsx (new — light block)

| Field | Detail |
|---|---|
| Props | `{ stats: LandingContent['highlightStats'] }` |
| Local state | None |
| Behavior | Renders `publishedCaseStudyCount` and `lastDataRefresh` as a small stat row on `LandingPage` |
| Edge cases | `lastDataRefresh: null` (no pipeline run has ever completed) → render "Not yet available" rather than an invalid date string |
| Test file | `tests/components/content/HighlightStats.test.tsx` |

### src/components/content/DataSourceCard.tsx (new — light block)

| Field | Detail |
|---|---|
| Props | `{ source: MethodologyContent['dataSources'][number] }` |
| Local state | None |
| Behavior | One card per `dataSources` entry — renders `name` and `description`; a purely presentational mapping of `MethodologyPage`'s array, no per-card fetch |
| Edge cases | None |
| Test file | `tests/components/content/DataSourceCard.test.tsx` |

### src/components/content/LimitationsList.tsx (new — light block)

| Field | Detail |
|---|---|
| Props | `{ limitations: string[] }` |
| Local state | None |
| Behavior | Renders every entry in `limitations` as a plain, always-visible list item — never behind an accordion, "read more," or collapsed-by-default control, per frontend spec 3.4's transparency-first positioning and the NFR requiring the tool not be mistaken for a guarantee |
| Edge cases | Empty array — renders nothing (not an empty bullet or a "no limitations" claim); should not occur in practice since methodology content is admin-authored, but the component does not assume a non-empty array |
| Test file | `tests/components/content/LimitationsList.test.tsx` |

### src/components/content/OnboardingOverlay.tsx (new — light block)

| Field | Detail |
|---|---|
| Props | `{ onDismiss: () => void }` |
| Local state | None — dismissal state (has this session already seen it) is owned by the parent (`GrowthMapPage`, per frontend spec 3.1's routing note), not by this component |
| Behavior | Static copy explaining map colors/scores (FR-13); no API call — this is the one component in the Site Content feature that is not backed by `useContent.ts`. Rendered from `GrowthMapPage` (Growth Map feature) but owned structurally here, per frontend spec 3.1 |
| Edge cases | Dismissal is session-only (e.g., a module-level flag or `sessionStorage`-equivalent in-memory state per the parent's implementation) — never persisted across browser sessions, so a returning visitor in a new session sees it again, per UC-09's "first-time visitor" framing |
| Test file | `tests/components/content/OnboardingOverlay.test.tsx` |

---

### src/pages/LandingPage.tsx (new)

| Field | Detail |
|---|---|
| Purpose | Introduces the tool before the visitor enters the map (UC-07). |
| Route guard + layout | Public, `PublicLayout` |
| Local state | None |
| Behavior | 1. `useLandingContent()`. 2. Renders `tagline`, `intro`, and `HighlightStats`. 3. Renders a clear entry-point CTA linking to `ROUTES.MAP`, per FR-11. |
| Components | HighlightStats |

**States:** loading (skeleton) · error (inline + retry — the landing page must still render its static CTA into the map even if `highlightStats` fails to load, since entering the map must never be blocked by this feature) · success

---

### src/pages/MethodologyPage.tsx (new)

| Field | Detail |
|---|---|
| Purpose | Explains the composite score, data sources, validation approach, and limitations in plain language (UC-06). |
| Route guard + layout | Public, `PublicLayout` |
| Local state | None |
| Behavior | 1. `useMethodologyContent()`. 2. Renders `scoreExplanation` as prose. 3. Renders one `DataSourceCard` per `dataSources` entry. 4. Renders `validationApproach` as prose. 5. Renders `LimitationsList` with `limitations`. |
| Components | DataSourceCard, LimitationsList |

**States:** loading (skeleton) · error (inline + retry — this is a static informational page per UC-06's alternate flow, so a load failure is a plain error state, not a partial/degraded render) · success

---

**Next:** proceed to → [9-5. Frontend: Admin Access]
