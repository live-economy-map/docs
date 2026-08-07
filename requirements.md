## Project: Shadow Economy Map — Addis Ababa Growth Detection MVP
**Links back to:** [1. Problem & Solution Statement]

---

### 2.1 Functional Requirements

Requirements are grouped and numbered in the order a real user moves through the system: anonymous visitor (map exploration, no account needed) → admin authentication (required gate for data management only) → admin/data-team actions → cross-cutting behavior. There is no customer account, cart, or checkout anywhere in this system — the MVP has no e-commerce component; it is a public data-exploration tool with a protected admin/data-management area.

#### A. Visitor / Public User (no login required)

| ID | Requirement | Priority | Linked Use Case |
|---|---|---|---|
| FR-01 | A visitor can view the interactive Addis Ababa map showing the composite growth score per grid cell without logging in | Must | UC-01 |
| FR-02 | A visitor can use a time-slider to move through available time periods and observe how the growth score for the map changes over time | Must | UC-02 |
| FR-03 | A visitor can click or tap a grid cell to open a detail panel showing that cell's composite score | Must | UC-03 |
| FR-04 | The grid cell detail panel shows the individual signal breakdown contributing to the composite score (night-time light trend, built-up area change, relative wealth estimate, related news activity) | Must | UC-03 |
| FR-05 | A visitor can toggle between the composite growth-score heatmap view and raw underlying layers (e.g., raw night-light intensity, raw built-up area) | Should | UC-04 |
| FR-06 | A visitor can view a small set of marked case-study locations on the map, each representing an independently verified known-growth area | Must | UC-05 |
| FR-07 | Clicking a case-study marker shows supporting evidence (e.g., a news reference or report date) alongside the date the growth score began rising | Must | UC-05 |
| FR-08 | A visitor can view a before/after satellite or built-up-area comparison for at least one case-study location | Should | UC-05 |
| FR-09 | A visitor can view a short "About / Methodology" page explaining what the composite score means, which data sources feed it, and its known limitations | Must | UC-06 |
| FR-10 | The map defaults to the most recent available time period on first load | Should | UC-01 |

#### B. Admin Authentication & Access

| ID | Requirement | Priority | Linked Use Case |
|---|---|---|---|
| FR-11 | The project/data team can log in to a protected admin area not accessible to public visitors | Must | UC-07 |
| FR-12 | An invalid admin login (wrong username/password) returns a clear error without revealing which field was incorrect | Must | UC-07 |
| FR-13 | Admin sessions expire after a period of inactivity | Should | UC-07 |

#### C. Admin / Data Pipeline Management

| ID | Requirement | Priority | Linked Use Case |
|---|---|---|---|
| FR-14 | An admin can trigger a data refresh pull for a given source (VIIRS, GHSL, Meta RWI, GDELT) for the Addis Ababa bounding box | Must | UC-08 |
| FR-15 | An admin can view the last successful update timestamp for each data source | Must | UC-08 |
| FR-16 | An admin can view a status/health indicator per source (e.g., last pull succeeded, failed, or is stale beyond an expected threshold) | Must | UC-08 |
| FR-17 | An admin can trigger a recomputation of the composite growth score after new source data is pulled | Must | UC-09 |
| FR-18 | An admin can view and adjust the relative weighting of each of the four data sources in the composite score formula, for experimentation and tuning during methodology development | Should | UC-09 |
| FR-19 | An admin can add, edit, or remove a case-study location (name, coordinates, supporting evidence link/description, confirmed date) | Must | UC-10 |
| FR-20 | An admin can view a log of past data pipeline runs, including timestamp, source, and success/failure status | Should | UC-08 |

#### D. Cross-Cutting

| ID | Requirement | Priority | Linked Use Case |
|---|---|---|---|
| FR-21 | All public-facing pages (map, case studies, methodology) render and function correctly on mobile screen sizes | Must | UC-11 |
| FR-22 | The admin area renders and functions correctly on standard desktop screen sizes; mobile support for admin is not required | Won't (MVP) | UC-11 |
| FR-23 | Map interactions (pan, zoom, time-slider, cell selection) respond without a full page reload | Must | UC-01 |

Numbering convention: FR-01, FR-02... never renumber once the project moves past this initial draft — deprecate with a strikethrough note instead, so future references (issues, methodology write-ups) don't silently point at the wrong requirement.

---

### 2.2 Non-Functional Requirements

| Category | Requirement | How it's verified |
|---|---|---|
| Performance | The map view (initial load, one grid layer) renders in under 3 seconds on a standard broadband connection | Manual timing / browser performance audit |
| Performance | Switching the time-slider updates the visible map layer in under 1.5 seconds | Manual timing |
| Security | The admin login is authenticated and passwords are hashed (e.g., bcrypt); never logged or returned in any response | Code review checklist |
| Security | The admin area is inaccessible to unauthenticated users, including via direct URL access | Manual spot check |
| Security | Repeated failed admin login attempts are rate-limited to reduce brute-force risk | Manual spot check |
| Availability | The public map is available on a standard best-effort basis appropriate for an academic MVP; no formal SLA is required | Manual uptime spot checks |
| Scalability | The data pipeline and map can support the full Addis Ababa grid (city-wide coverage) at the chosen cell resolution without redesign | Reviewed at design time |
| Scalability | The system architecture does not preclude adding a second city's data in the future, even if not built in this release | Reviewed at design time |
| Accessibility | The map, time-slider, and case-study panels are usable via touch and keyboard, with readable contrast and text sizing | Manual spot check |
| Data accuracy / provenance | Every displayed composite score and case study clearly indicates which data sources and time period it is derived from | Design/UX review |
| Observability | Data pipeline runs (success, failure, timestamp, source) are logged for later review | Code review checklist |
| Data retention | Historical composite score snapshots are retained (not overwritten) so the time-slider can show genuine historical change, not just the latest state | Code review checklist |
| Usability | The methodology and limitations are presented in plain language accessible to a non-technical viewer (e.g., an NGO or planning-office reader), not just a technical audience | Design review against Section 1 success criteria |
| Usability | Visual design clearly distinguishes "detected growth signal" from "confirmed/validated case study" so the tool cannot be mistaken for a prediction or guarantee | Design/UX review |
| Reproducibility | The methodology for computing the composite score is documented in enough detail that a reviewer/committee member could, in principle, reproduce a given cell's score from raw source data | Documentation review |

---

### 2.3 Explicitly Out of Requirements (things people might assume, but aren't required)

- No forecasting or prediction of future growth, prices, or infrastructure needs — the MVP detects current/recent signals only
- No coverage beyond Addis Ababa — a second city is a stretch goal only, not a requirement
- No user accounts, registration, or login for public visitors — the map is fully public and anonymous
- No use of business-registration data as a model input or ground truth, due to unreliable bulk access in Ethiopia
- No use of traffic data, social media trend data, or weather data as model inputs for this release
- No use of OpenStreetMap data as a core scoring input — supplementary/contextual display only, if used at all
- No real-time or continuously live data updates — periodic, admin-triggered refreshes are sufficient
- No multi-segment commercial features (subscriptions, premium reports, paid API access, billing) — this is a research/demonstration MVP, not a monetized product
- No role-based access control beyond a single admin login — no multiple staff accounts or permission tiers
- No mobile support requirement for the admin area — only the public-facing pages must be mobile-responsive
- No automated alerting (e.g., notifying users when a new area starts showing growth signals) — the MVP is explore-on-demand only

**Next:** proceed to → [3. Use Cases]
