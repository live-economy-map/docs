## Project: Shadow Economy Map — Frontend Test Documentation: Site Content & Onboarding
**Links back to:** [9-4. Frontend Function-Level Spec: Site Content & Onboarding], [9-1. Frontend Test Documentation: Shared/Config], [9-2. Frontend Test Documentation: Growth Map & Exploration]

Per the Frontend Testing Guide: hooks mock `@/lib/axios`; pages mock this feature's own hooks (`@/hooks/useContent`). `OnboardingOverlay` is the one component in this module with no API dependency of its own — it lives in `src/components/content/` and is documented here, but is *rendered* inside `GrowthMapPage` (Growth Map feature), matching the cross-feature pattern already established for `CaseStudyMarker` in 9-3. Its own test file has no hook to mock; `GrowthMapPage.test.tsx` (9-2) treats it as an opaque child.

---

### 9.1 Test File Map

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| `src/hooks/useContent.ts` | `tests/hooks/useContent.test.ts` | Unit (`renderHook`, mocked `@/lib/axios`) | ☐ |
| `src/components/content/HighlightStats.tsx` | `tests/components/content/HighlightStats.test.tsx` | Component (RTL, presentational) | ☐ |
| `src/components/content/OnboardingOverlay.tsx` | `tests/components/content/OnboardingOverlay.test.tsx` | Component (RTL, presentational, static copy) | ☐ |
| `src/components/content/DataSourceCard.tsx` | `tests/components/content/DataSourceCard.test.tsx` | Component (RTL, presentational) | ☐ |
| `src/components/content/LimitationsList.tsx` | `tests/components/content/LimitationsList.test.tsx` | Component (RTL, presentational) | ☐ |
| `src/pages/LandingPage.tsx` | `tests/pages/LandingPage.test.tsx` | Component (RTL, mocked `useLandingContent`) | ☐ |
| `src/pages/MethodologyPage.tsx` | `tests/pages/MethodologyPage.test.tsx` | Component (RTL, mocked `useMethodologyContent`) | ☐ |

---

### 9.2 Test Case Detail — useContent.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| useLandingContent calls the correct endpoint | mocked `api.get` resolves a `LandingContent` shape | `renderHook(() => useLandingContent())` | `api.get` called with `('/content/landing')` |
| useLandingContent has no params in its query key | inspect cache | render | cache entry keyed exactly `[QUERY_KEYS.CONTENT_LANDING]` |
| useLandingContent uses a 5-minute staleTime | inspect the query's options (or the `QueryClient` cache entry's config) | render | `staleTime` equals `5 * 60 * 1000`, per the hook's documented configuration — landing copy changes rarely enough not to refetch on every mount |
| useMethodologyContent calls the correct endpoint | mocked `api.get` resolves a `MethodologyContent` shape | `renderHook(() => useMethodologyContent())` | `api.get` called with `('/content/methodology')` |
| dataSources array passes through unmodified, GDELT included | mocked `api.get` resolves `dataSources` with all 4 entries (`VIIRS`, `GHSL`, `RWI`, `GDELT`) | render, await | `result.current.data.dataSources` has 4 entries including GDELT — this hook has no exclusion logic; GDELT's exclusion is specific to `LayerToggle` (9-2) and `getRawSignalLayer`, not to methodology content |
| Error state on failure | mocked `api.get` rejects | render, await | `result.current.isError === true` for either hook |

---

### 9.2 Test Case Detail — HighlightStats.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders publishedCaseStudyCount | props: `{publishedCaseStudyCount: 3, lastDataRefresh: '2026-08-01T00:00:00Z'}` | render | the count `3` is visible |
| Renders a formatted lastDataRefresh | same props | render | a human-readable date derived from the ISO string is visible (not the raw ISO string verbatim) |
| Renders a fallback when lastDataRefresh is null | props: `{publishedCaseStudyCount: 0, lastDataRefresh: null}` | render | a "no data yet"-equivalent fallback shown instead of attempting to format `null` as a date (which would otherwise render "Invalid Date") |

---

### 9.2 Test Case Detail — OnboardingOverlay.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders static legend/onboarding copy when shown | props: `{visible: true, onDismiss: vi.fn()}` | render | onboarding text explaining map colors/scores is visible |
| Renders nothing when not visible | props: `{visible: false, onDismiss: vi.fn()}` | render | overlay content NOT present |
| Dismissing calls onDismiss | props: `{visible: true, onDismiss: vi.fn()}` | `userEvent.click` the dismiss control | `onDismiss` called |
| Makes no API call of its own | no hook mocked/imported for this test at all | render | component renders synchronously from props alone — confirms this component has no data dependency, consistent with API spec 3.1's note that this copy ships with the frontend, not the API |

---

### 9.2 Test Case Detail — DataSourceCard.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders name and description | props: `{key: 'VIIRS', name: 'VIIRS Night-Time Lights', description: '...'}` | render | name and description text visible |
| Renders a GDELT card identically to any other source | props: `{key: 'GDELT', name: 'GDELT', description: 'string — explains its validation-only role...'}` | render | renders through the same code path as any other source — no special-cased branch skips or alters GDELT's card, since its distinct role is conveyed entirely through the description copy the backend provides, not frontend logic |

---

### 9.2 Test Case Detail — LimitationsList.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders one item per limitation | props: `limitations: [5 strings]` | render | 5 list items rendered, one per string, text matching |
| Renders nothing extraneous for an empty array | props: `limitations: []` | render | no list items rendered; component does not throw on an empty array |

---

### 9.3 Test Case Detail — LandingPage.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders tagline and intro on success | `vi.mock('@/hooks/useContent')` → mocked `useLandingContent` success with `{tagline, intro, highlightStats}` | render `<LandingPage />` | tagline and intro text visible |
| Renders HighlightStats with the fetched stats | mocked hook success | render | `HighlightStats` present, receiving the mocked `highlightStats` object |
| Loading state | mocked hook → `{isPending: true}` | render | loading indicator shown, no tagline/intro yet |
| Error state | mocked hook → `{isError: true}` | render | an error/fallback message shown, not a blank page |
| Provides an entry point into the map | mocked hook success | render | a link/button to `ROUTES.MAP` is present |

---

### 9.3 Test Case Detail — MethodologyPage.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders scoreExplanation and validationApproach | mocked `useMethodologyContent` success with full `MethodologyContent` | render `<MethodologyPage />` | both text blocks visible |
| Renders one DataSourceCard per dataSources entry, GDELT included | mocked hook success with 4 `dataSources` entries | render | 4 `DataSourceCard`s rendered, one of them representing GDELT — this page is precisely where GDELT's validation-only role is explained to a visitor, unlike the map's raw-layer toggle which excludes it entirely |
| Renders LimitationsList with the fetched limitations | mocked hook success | render | `LimitationsList` present with the given array |
| Loading state | mocked hook → `{isPending: true}` | render | loading indicator shown |
| Error state | mocked hook → `{isError: true}` | render | error/fallback message shown |

---

### 9.4 Coverage Honesty Check (per PR Steward, at review time)

- [ ] `HighlightStats`'s null-`lastDataRefresh` case actually renders and asserts a fallback string — not merely asserting the component doesn't throw, which would also pass if it silently rendered "Invalid Date"
- [ ] `DataSourceCard`'s GDELT case and `MethodologyPage`'s 4-source case both explicitly confirm GDELT is present and rendered through the *same* path as the other three — this is the one place in the whole frontend suite where GDELT inclusion is correct behavior, the inverse of every assertion in 9-2's `LayerToggle` tests, and a reviewer should treat an accidental exclusion here as seriously as an accidental inclusion there
- [ ] `useContent.test.ts`'s `staleTime` assertion actually inspects the real configured value (5 minutes), not a hardcoded literal in the test that happens to match by coincidence if the hook's config drifts
- [ ] `LandingPage`/`MethodologyPage` loading and error states are both tested as genuinely distinct render outputs from the success state — not implied by "the mock returns pending/error, so presumably something different shows"

---

### 9.5 Out of Scope for Automated Testing (and why)

- **Real copy/content accuracy** (whether `scoreExplanation` or `limitations` text is itself correct, complete, or non-technical enough for the NFR's "plain language" bar) — these tests verify the frontend renders whatever the backend returns; the actual wording is a content/design review concern (per Doc 2's Usability NFR), not a unit-test concern.
- **First-time-visitor detection logic for showing OnboardingOverlay** (e.g. a localStorage "has seen onboarding" flag) — this file tests the overlay component itself via its `visible`/`onDismiss` props; if `GrowthMapPage` (9-2) derives that visibility from browser storage, that derivation is tested there, not here, since this component takes visibility as an external prop and has no storage logic of its own.
- **Real date-locale formatting edge cases** (timezones, non-English locales) — `HighlightStats`'s formatting test confirms a formatted string is shown, not the exact locale-specific output for every possible browser/timezone combination.

---

**Next:** proceed to → [9-5. Frontend Test Documentation: Admin Access]
