## Project: Shadow Economy Map — Addis Ababa Growth Detection MVP
**Links back to:** [1. Problem & Solution Statement], [2. Functional & Non-Functional Requirements]

Each use case maps to at least one Functional Requirement and, later, at least one API endpoint.

---

### A. Visitor / Public Map Exploration

#### UC-01: Browse the growth-score map

| Field | Detail |
|---|---|
| Actor | Anonymous visitor |
| Precondition | None — visitor has opened the site |
| Trigger | Visitor lands on the map page |
| Linked FR | FR-01, FR-10, FR-23 |

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

**Postcondition (success):** Visitor sees the map updated to the selected time period.

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
3. System displays the composite score and the individual signal breakdown (night-light trend, built-up area change, relative wealth estimate, news signal).

**Alternate / error flows:**
- If the cell has no data for the selected period (e.g., outside data coverage): system displays a "no data available for this cell/period" message instead of a score.

**Postcondition (success):** Visitor understands both the composite score and what is driving it for that specific location and time.

---

#### UC-04: Toggle between composite view and raw data layers

| Field | Detail |
|---|---|
| Actor | Anonymous visitor |
| Precondition | Visitor is viewing the map |
| Trigger | Visitor selects a different layer (e.g., raw night-lights, raw built-up area) from a layer toggle |
| Linked FR | FR-05 |

**Main flow:**
1. Visitor selects a raw data layer instead of the composite score layer.
2. System swaps the map's shading to reflect the selected raw layer for the current time period.
3. Visitor can switch back to the composite view at any time.

**Alternate / error flows:**
- If a raw layer has no data for the selected period: system displays an empty-state message for that layer only, without affecting other layers.

**Postcondition (success):** Visitor can see the underlying raw signal, not just the derived composite score, supporting transparency about how the score is built.

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
2. System displays the location's supporting evidence (e.g., news reference, report date) and the date its growth score began rising.
3. If available, system displays a before/after comparison (satellite or built-up-area imagery) for that location.

**Alternate / error flows:**
- If no before/after imagery is available for that specific case study: system displays the score and evidence only, without the comparison view.

**Postcondition (success):** Visitor can verify that a detected growth signal corresponds to a real, independently confirmed outcome.

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
2. System displays an explanation of the composite score, the four data sources used, and the tool's known limitations (single-city scope, detection rather than prediction, coarse resolution of some sources).

**Alternate / error flows:**
- None — this is a static informational page.

**Postcondition (success):** Visitor understands what the tool does and does not claim to do.

---

### B. Admin Authentication

#### UC-07: Admin logs into the protected area

| Field | Detail |
|---|---|
| Actor | Project/data team member (admin) |
| Precondition | Admin has valid credentials |
| Trigger | Admin navigates to the admin login screen and submits credentials |
| Linked FR | FR-11, FR-12, FR-13 |

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

#### UC-08: Refresh source data

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | Admin is authenticated in the admin area |
| Trigger | Admin triggers a data pull for one or more sources |
| Linked FR | FR-14, FR-15, FR-16, FR-20 |

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

#### UC-09: Recompute the composite growth score

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | Admin is authenticated; at least one source has updated data available | 
| Trigger | Admin triggers recomputation, or it runs automatically after a successful refresh |
| Linked FR | FR-17, FR-18 |

**Main flow:**
1. Admin triggers recomputation of the composite score for the affected time period.
2. System recalculates the composite score per grid cell using the current source-weighting configuration.
3. System stores the new score snapshot without overwriting prior historical snapshots (so the time-slider continues to reflect true history).
4. Updated scores become visible on the public map.

**Alternate / error flows:**
- If recomputation fails partway (e.g., missing data for one source in one area): system flags affected cells as incomplete for that period rather than displaying a misleading score.

**Postcondition (success):** The public map reflects an updated, correctly weighted composite score.

---

#### UC-10: Manage case studies

| Field | Detail |
|---|---|
| Actor | Admin |
| Precondition | Admin is authenticated in the admin area |
| Trigger | Admin adds, edits, or removes a case-study entry |
| Linked FR | FR-19 |

**Main flow (add/edit):**
1. Admin submits or updates a case study's name, coordinates, supporting evidence description/link, and confirmed date.
2. System validates input and publishes/updates the case study, making it visible on the public map.

**Main flow (remove):**
1. Admin selects a case study and removes it.
2. System hides it from the public map immediately.

**Alternate / error flows:**
- If required fields are missing (e.g., no coordinates or no supporting evidence): system returns a validation error and does not publish the entry.

**Postcondition (success):** The public-facing case studies accurately reflect the project team's verified findings.

---

### D. Cross-Cutting

#### UC-11: Use the public map on a mobile device

| Field | Detail |
|---|---|
| Actor | Anonymous visitor |
| Precondition | Visitor is using a mobile screen size/touch device |
| Trigger | Visitor performs any of the public flows above (UC-01 through UC-06) on a mobile device |
| Linked FR | FR-21 |

**Main flow:**
1. Visitor loads the public map on a mobile screen.
2. System renders a mobile-responsive layout — map, time-slider, cell detail panel, and case studies all remain usable via touch.
3. Visitor completes their intended exploration without loss of functionality compared to desktop.

**Alternate / error flows:**
- None — this is a rendering/behavior constraint applied across all public-facing use cases, not a distinct error-prone flow of its own. Note: per FR-22, this constraint applies to the public map only — the admin area is desktop-only for this release.

**Postcondition (success):** All public-facing functionality remains available and usable at mobile screen sizes.

---

**Next:** proceed to → [4. Database & Data Model / Architecture]
