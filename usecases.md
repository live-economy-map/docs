## Project: Shadow Economy Map — Addis Ababa Growth Detection MVP (V1)
**Links back to:** [1. Problem & Solution Statement], [2. Functional & Non-Functional Requirements], [roadmap.md]

Each use case maps to at least one Functional Requirement and, later, at least one API endpoint. Use cases are numbered sequentially in the order they appear below, matching the order a real user (visitor, then admin) moves through the system.

---

### A. Visitor / Public Map Exploration

#### UC-01: Browse the growth-score map

| Field | Detail |
|---|---|
| Actor | Anonymous visitor |
| Precondition | None — visitor has opened the site |
| Trigger | Visitor lands on the map page |
| Linked FR | FR-01, FR-10, FR-28 |

**Main flow:**
1. Visitor opens the site.
2. System loads the Addis Ababa map with the composite growth-score layer for the most recent available time period.
3. Visitor pans and zooms the map.
4. System renders grid cells shaded by their composite growth score.

**Alternate / error flows:**
- If no processed data exists yet for the default period: system displays an empty-state message and, if available, falls back to the most recent period that does have data.

**Postcondition (success):** Visitor sees the current growth-score map and can proceed to explore time, cell detail, or case studies.

---

#### UC-02: Move through time using the time-slider

| Field | Detail |
|---|---|
| Actor | Anonymous visitor |
| Precondition | Visitor is viewing the map |
| Trigger | Visitor drags or steps the time-slider to a different period |
| Linked FR | FR-02 |

**Main flow:**
1. Visitor moves the time-slider to a selected period.
2. System requests the composite score layer for that period.
3. System updates the map to reflect that period's scores without a full page reload.

**Alternate / error flows:**
- If no data exists for the selected period: system displays the nearest available period and indicates the substitution to the visitor.

**Postcondition (success):** Visitor sees the map updated to the selected time period. The slider never offers a period beyond the present — it replays history, it does not project forward.

---

#### UC-03: Inspect a single grid cell

| Field | Detail |
|---|---|
| Actor | Anonymous visitor |
| Precondition | Visitor is viewing the map with a loaded data period |
| Trigger | Visitor clicks or taps a grid cell |
| Linked FR | FR-03, FR-04 |

**Main flow:**
1. Visitor clicks a grid cell.
2. System opens a detail panel for that cell and period.
3. System displays: the composite score, the raw signal breakdown (night-light trend, built-up area change, relative wealth estimate), a score trend arrow vs. the previous period, a mini sparkline of the cell's score history, the cell's approximate location label, and a data-completeness/freshness indicator.
4. System requests and displays an AI-generated plain-language summary of the cell's signals, shown once available (numeric data in step 3 displays immediately and does not wait on the AI response).

**Alternate / error flows:**
- If the cell has no data for the selected period (e.g., outside data coverage): system displays a "no data available for this cell/period" message instead of a score.
- If the AI summary fails to generate or times out: the panel still displays all numeric data normally; the summary section shows a brief "summary unavailable" note instead of blocking the rest of the panel.

**Postcondition (success):** Visitor understands both the composite score and what is driving it for that specific location and time, in both numeric and plain-language form.

---

#### UC-04: Toggle between composite view and raw data layers

| Field | Detail |
|---|---|
| Actor | Anonymous visitor |
| Precondition | Visitor is viewing the map |
| Trigger | Visitor selects a different layer from a layer toggle |
| Linked FR | FR-05 |

**Main flow:**
1. Visitor selects one of the 3 raw data layers (night-time lights, built-up area, relative wealth) instead of the composite score layer.
2. System swaps the map's shading to reflect the selected raw layer for the current time period.
3. Visitor can switch back to the composite view at any time.

**Alternate / error flows:**
- If a raw layer has no data for the selected period: system displays an empty-state message for that layer only, without affecting other layers.

**Postcondition (success):** Visitor can see the underlying raw signal, not just the derived composite score, supporting transparency about how the score is built. Note: GDELT is intentionally not offered here, since it is discrete event data rather than a continuous per-cell value — it appears via case-study markers (UC-05) instead.

---

#### UC-05: View a validated case study

| Field | Detail |
|---|---|
| Actor | Anonymous visitor |
| Precondition | At least one case-study location has been published by an admin |
| Trigger | Visitor clicks a case-study marker on the map |
| Linked FR | FR-06, FR-07, FR-08 |

**Main flow:**
1. Visitor clicks a case-study marker.
2. System displays the location's supporting evidence (e.g., news reference, report date), the date its growth score began rising, and the date growth was independently confirmed.
3. If available, system displays a before/after comparison (satellite or built-up-area imagery) for that location.

**Alternate / error flows:**
- If no before/after imagery is available for that specific case study: system displays the score and evidence only, without the comparison view.

**Postcondition (success):** Visitor can verify that a detected growth signal corresponds to a real, independently confirmed outcome, and can see that the score's rise date is not later than the confirmation date.

---

#### UC-06: View methodology and limitations

| Field | Detail |
|---|---|
| Actor | Anonymous visitor |
| Precondition | None |
| Trigger | Visitor navigates to the "About / Methodology" page |
| Linked FR | FR-09 |

**Main flow:**
1. Visitor navigates to the methodology page.
2. System displays an explanation of the composite score, the four data sources used, how case studies are validated, and the tool's known limitations (single-city scope, detection rather than prediction, coarse resolution of some sources).

**Alternate / error flows:**
- None — this is a static informational page.

**Postcondition (success):** Visitor understands what the tool does and does not claim to do.

---

#### UC-07: View the landing page

| Field | Detail |
|---|---|
| Actor | Anonymous visitor |
| Precondition | None |
| Trigger | Visitor navigates to the site's root URL |
| Linked FR | FR-11 |

**Main flow:**
1. Visitor lands on the homepage.
2. System displays a brief introduction to the tool and a clear entry point into the map.

**Alternate / error flows:**
- None — static informational page.

**Postcondition (success):** Visitor understands what the tool is before entering the interactive map.

---

#### UC-08: Browse the case studies list page

| Field | Detail |
|---|---|
| Actor | Anonymous visitor |
| Precondition | At least one case study has been published |
| Trigger | Visitor navigates to the Case Studies page |
| Linked FR | FR-12 |

**Main flow:**
1. Visitor navigates to the Case Studies page.
2. System displays all published case studies as a list, independent of the map view.
3. Visitor selects one to view its full evidence detail (same content as UC-05).

**Alternate / error flows:**
- If no case studies are published yet: system displays an empty-state message.

**Postcondition (success):** Visitor can review all validation evidence without needing to locate markers on the map.

---

#### UC-09: See the map legend/onboarding overlay

| Field | Detail |
|---|---|
| Actor | Anonymous visitor (first-time) |
| Precondition | Visitor has not previously dismissed the overlay in this session |
| Trigger | Visitor loads the map for the first time in a session |
| Linked FR | FR-13 |

**Main flow:**
1. Visitor loads the map.
2. System displays a brief overlay explaining what the map colors and scores mean.
3. Visitor dismisses the overlay and proceeds to explore the map.

**Alternate / error flows:**
- None.

**Postcondition (success):** Visitor has a baseline understanding of how to read the map before exploring it.

---

#### UC-10: Search the map using natural language

| Field | Detail |
|---|---|
| Actor | Anonymous visitor |
| Precondition | Visitor is viewing the map |
| Trigger | Visitor types a plain-language query into the search bar |
| Linked FR | FR-14 |

**Main flow:**
1. Visitor types a query (e.g., "show me areas near Bole with rising construction").
2. System parses the query into map filter parameters (location, time range, signal type) using an AI/LLM call.
3. System applies the resulting filters to the map view.

**Alternate / error flows:**
- If the query cannot be confidently parsed into valid filters: system informs the visitor it couldn't understand the query and leaves the current map view unchanged, rather than guessing.

**Postcondition (success):** The map reflects the visitor's intended query without them needing to manually operate each filter control.

---

### B. Admin Authentication

#### UC-11: Admin logs into the protected area

| Field | Detail |
|---|---|
| Actor | Project/data team member (admin) |
| Precondition | Admin has valid credentials |
| Trigger | Admin navigates to the admin login screen and submits credentials |
| Linked FR | FR-15, FR-16, FR-17 |

**Main flow:**
1. Admin submits username/email and password on the login screen.
2. System validates credentials.
3. System issues an admin session and grants access to the admin area.

**Alternate / error flows:**
- If credentials are invalid: system returns a generic error without revealing which field was incorrect.
- If a session has been inactive beyond the defined timeout: system requires re-authentication.

**Postcondition (success):** Admin has access to the protected data-management area.

---

### C. Admin / Data Pipeline Management

#### UC-12: Refresh source data

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | Admin is authenticated in the admin area |
| Trigger | Admin triggers a data pull for one or more sources |
| Linked FR | FR-18, FR-19, FR-20, FR-24 |

**Main flow:**
1. Admin selects a data source (VIIRS, GHSL, Meta RWI, or GDELT) and triggers a refresh for the Addis Ababa bounding box.
2. System pulls the latest available data for that source.
3. System updates the "last successful update" timestamp and health status for that source.
4. System logs the run (timestamp, source, success/failure).

**Alternate / error flows:**
- If the pull fails (e.g., source API unavailable): system marks the source status as failed, logs the failure, and leaves prior data untouched.
- If the pulled data is malformed or incomplete: system flags the source as stale/needs attention rather than silently accepting bad data.

**Postcondition (success):** The selected source's underlying data is up to date and ready for score recomputation.

---

#### UC-13: Recompute the composite growth score

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | Admin is authenticated; at least one source has updated data available |
| Trigger | Admin triggers recomputation, or it runs automatically after a successful refresh |
| Linked FR | FR-21, FR-22 |

**Main flow:**
1. Admin triggers recomputation of the composite score for the affected time period.
2. System recalculates the composite score per grid cell using the current source-weighting configuration.
3. System stores the new score snapshot without overwriting prior historical snapshots (so the time-slider continues to reflect true history).
4. Updated scores become visible on the public map.

**Alternate / error flows:**
- If recomputation fails partway (e.g., missing data for one source in one area): system flags affected cells as incomplete for that period rather than displaying a misleading score.

**Postcondition (success):** The public map reflects an updated, correctly weighted composite score, traceable to the specific weight configuration used.

---

#### UC-14: Manage case studies

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | Admin is authenticated in the admin area |
| Trigger | Admin adds, edits, or removes a case-study entry |
| Linked FR | FR-23 |

**Main flow (add/edit):**
1. Admin submits or updates a case study's name, coordinates, supporting evidence description/link, score-rise date, confirmed date, and optional before/after imagery.
2. System validates input and publishes/updates the case study, making it visible on the public map and case studies list page.

**Main flow (remove):**
1. Admin selects a case study and removes it.
2. System hides it from the public map and case studies list page immediately.

**Alternate / error flows:**
- If required fields are missing (e.g., no coordinates or no supporting evidence): system returns a validation error and does not publish the entry.

**Postcondition (success):** The public-facing case studies accurately reflect the project team's verified findings.

---

#### UC-15: AI-assisted case-study candidate discovery (internal)

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | Admin is authenticated in the admin area |
| Trigger | Admin runs the candidate-discovery tool for a region or keyword |
| Linked FR | FR-25 |

**Main flow:**
1. Admin provides a search focus (e.g., a neighborhood name or general area).
2. System uses an AI/LLM-assisted search to surface recent news mentions suggesting growth or development activity in that area.
3. Admin reviews the surfaced candidates and manually verifies each one against the tiered evidence process before publishing any as a case study (UC-14).

**Alternate / error flows:**
- If no relevant candidates are found: system indicates no results, rather than fabricating a plausible-sounding but unverified suggestion.

**Postcondition (success):** The admin has a faster starting point for case-study research; no candidate is published without manual human verification. This is an internal workflow tool, not a public-facing feature.

---

### D. Cross-Cutting

#### UC-16: Use the public site on a mobile device

| Field | Detail |
|---|---|
| Actor | Anonymous visitor |
| Precondition | Visitor is using a mobile screen size/touch device |
| Trigger | Visitor performs any of the public flows above (UC-01 through UC-10) on a mobile device |
| Linked FR | FR-26 |

**Main flow:**
1. Visitor loads any public page on a mobile screen.
2. System renders a mobile-responsive layout — map, time-slider, cell detail panel, case studies, and other public pages all remain usable via touch.
3. Visitor completes their intended exploration without loss of functionality compared to desktop.

**Alternate / error flows:**
- None — this is a rendering/behavior constraint applied across all public-facing use cases, not a distinct error-prone flow of its own. Note: per FR-27, this constraint applies to public pages only — the admin area is desktop-only for V1.

**Postcondition (success):** All public-facing functionality remains available and usable at mobile screen sizes.

---

**Next:** proceed to → [4. Database & Data Model / Architecture]
