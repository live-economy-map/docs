## 1. Problem & Solution Statement

**Project:** Shadow Economy Map

**Owners:**

| No. | Name | Id |
|---|---|---|
| 1 | Adonay Chilotu | CTC-1611-26 |
| 2 | Abrham Teramed | CTC-3158-26 |
| 3 | Amira Niguse | CTC-2988-26 |
| 4 | Aneni Kidanu | CTC-1451-26 |
| 5 | Abenezer Getamesay | CTC-2279-26 |
| 6 | Anteneh Wondwosen | CTC-286-26 |

**Status:** V1 (MVP) — Finalized for Implementation

---

### 1.1 Problem Statement

Ethiopia's economic growth is largely invisible to the tools that would normally track it.

Formal economic indicators — business registrations, tax records, census data — are slow, incomplete, or difficult to access at a granular level. Digitization efforts exist but are recent and not yet reliable enough to serve as clean, bulk data sources. Meanwhile, real economic activity — new construction, population inflows, rising commercial density — is happening now, especially in fast-urbanizing areas around Addis Ababa.

Banks, investors, real estate developers, and government planners making decisions about where to invest, lend, or build infrastructure are working with outdated or incomplete pictures of where growth is actually happening, particularly outside a handful of well-documented central districts.

The core gap: by the time official statistics confirm a neighborhood is growing, the opportunity to act on it early has already passed. Areas outside the best-mapped, wealthiest zones are systematically under-visible in the data that does exist, which biases decision-making toward places that are already well known and away from places that may need attention or investment most.

---

### 1.2 Target Users

| User type | Who they are | What they need from this system |
|---|---|---|
| Analyst / Researcher | A student, NGO staff member, or planning-office researcher exploring economic trends | View the interactive map, inspect growth scores by area, compare signals over time |
| Development / Planning Organization (illustrative customer) | An NGO or government planning body assessing where to focus development attention | A credible, independently-verifiable view of where economic activity is accelerating, without waiting for census or registry cycles |
| Admin | The project/data team maintaining the platform | Manage data pipeline runs, refresh source data, curate case studies, and monitor scoring outputs, without manual reprocessing for each update |

*(Future actors — not part of V1): Bank/investment analyst, real estate market researcher, government infrastructure planner, registered end customer (see Version Roadmap, 1.8).*

---

### 1.3 Proposed Solution

#### Grand Vision

A geospatial growth-signal platform that detects and, eventually, predicts emerging economic activity across Ethiopia, using satellite, mobility, and public data to surface growth before it appears in official statistics. At full scale, this platform would cover the whole country, integrate an evolving mix of data sources, forecast future hotspots rather than only detecting current activity, and serve paying customers through a live platform with subscriptions, API access, and custom reports. See **1.8 Version Roadmap** for the staged path toward this.

What customers would actually pay for, at full scale, is never the free public map — it's analysis run against a specific question:
- **NGOs / development orgs:** custom growth-ranking reports; targeted assessment of a named region
- **Government planning offices:** infrastructure-gap analysis (rising-activity areas with low existing infrastructure)
- **Real estate / investors:** early-signal alerts; due-diligence add-on reports for specific locations
- **Banks:** custom risk/opportunity scoring for lending decisions in specific neighborhoods

This is the long-term direction the project is working toward. It is not the version being built or validated in this release.

#### V1 / MVP (This Release)

A geospatial growth-**detection** tool for Addis Ababa, built entirely on free, high-frequency, publicly accessible data.

Anyone can explore the interactive map without an account, browsing the composite growth score across grid cells and moving through time via a time-slider to see how signals have changed. Clicking a grid cell reveals the underlying signal breakdown — night-light trend, built-up area change, relative wealth estimate, and related news activity — along with an AI-generated plain-language summary of what the signals indicate. A natural-language search lets visitors query the map conversationally (e.g., "show me areas near Bole with rising construction"). A small set of known, independently-verifiable growth areas in Addis Ababa are marked directly on the map as case studies, each annotated with the date the growth score rose and the date that growth was independently confirmed — forming the project's core validation evidence. An admin-only area allows the project team to trigger data pipeline refreshes, manage case studies, and monitor source data health.

**On timing — an important clarification:** the MVP does not predict the future. The economic activity it detects (construction, rising light/wealth signals) has already happened on the ground; the tool simply surfaces the physical/satellite evidence of that activity faster than news coverage or official statistics get around to confirming it publicly. It is ahead of the *reporting*, never ahead of the *event*.

The MVP does not forecast future hotspots or prices, does not use business-registration data as ground truth, and is scoped to a single city rather than the whole country.

---

### 1.4 Why This Approach (vs. alternatives considered)

**Prediction vs. detection:** A full predictive model (forecasting future hotspots, six months or more out) was considered, matching the original project concept. It was set aside for this release because predictive claims require historical ground truth to train and validate against, and reliable business-registration or economic ground-truth data is not currently accessible in bulk for Ethiopia. Detection of current, ongoing growth — validated against independently verifiable public evidence — was chosen instead as the honestly provable version of the same idea. Forecasting remains a Grand Vision goal (V5, see 1.8) once better ground-truth data becomes available.

**National vs. single-city scope:** Nationwide coverage was considered, matching the original vision. It was set aside for this release because data quality and availability vary significantly across regions, and committing to national scope up front risked shallow, unvalidated coverage everywhere rather than a rigorously proven result in one place. Addis Ababa was chosen as the initial scope because it has the strongest data density across all selected sources.

**Broad data mix vs. four core sources:** The original concept proposed seven or more data sources. Several were set aside: traffic and business-registration data are not reliably accessible in Ethiopia at this time; social media and weather data provide weak or noisy signal relative to engineering effort. Four sources — VIIRS night-time lights, GHSL built-up area/population data, Meta's Relative Wealth Index, and GDELT news signals — were chosen because each is free, well-documented, and already purpose-built for this kind of economic-proxy use case. VIIRS, GHSL, and RWI each measure a different stage of the same growth process (activity, investment, and standing wealth respectively); GDELT serves a distinct role as independent, human-verified evidence used only for case-study validation, not as a scored map layer.

**Simple weighted composite vs. a trained ML model:** A weighted-sum formula was chosen over a machine-learning model because no reliable ground-truth training dataset exists for Ethiopia (the same gap that rules out forecasting), and because transparency is core to this project's credibility strategy — a weighted sum is fully inspectable, while an ML model would be a black box. This is the same class of method used by established composite indices (e.g., HDI), not a novel or unproven technique.

**Public, account-free access vs. authentication:** The public map requires no login because there is nothing user-specific to protect — no personal data, no paid content, no transactions — and gating it would add friction without adding security, undermining the project's transparency-first credibility strategy. Authentication is used only where real stakes exist: the admin area, which can alter data and published case studies.

**Multi-segment platform vs. one illustrative customer:** The original plan listed five customer segments with no clear entry point. For this release, one illustrative customer angle — development finance / NGO / government planning use — was chosen to make the "who would use this" case concrete and credible, since organizations of this kind already use comparable data (e.g., Meta's RWI is used by real development organizations). A full multi-segment commercial platform remains part of the Grand Vision.

**Full grand-vision build vs. a staged version roadmap:** Rather than treating "MVP" and "Grand Vision" as two disconnected points, the project is planned as a sequence of versions (V1–V5, see 1.8). This gives a concrete, presentable path from the current build to the full vision. However, full engineering focus remains on completing V1 to a high standard before any V2+ work begins — the roadmap is a planning and presentation artifact, not a parallel build track, since V1's core detection claim must be proven solid before anything (monetization, accounts, forecasting) is reasonably built on top of it.

---

### 1.5 Scope

**In scope for V1 (MVP):**
- Interactive map of Addis Ababa showing a composite growth score per grid cell, browsable without an account
- Time-slider to view how the growth score has changed over time
- Click-to-inspect panel per grid cell, showing: composite score, the 3 raw signal values (night-lights, built-up area, relative wealth), an AI-generated plain-language summary, a score trend arrow, a mini sparkline history, cell location label, and data-completeness/freshness indicators
- Raw layer toggle for the 3 continuous data sources (VIIRS, GHSL, RWI) — GDELT is represented via case-study markers, not a map layer, since it is discrete event data rather than a continuous grid value
- Natural-language map query (AI-assisted search parsing into map filters)
- Data pipeline integrating four sources: VIIRS, GHSL, Meta RWI, GDELT
- A small set (2–4) of marked, annotated case-study locations, sourced via a tiered evidence process (government/official announcements → market research reports → infrastructure coverage → local news), each with before/after imagery where available
- AI-assisted case-study candidate discovery as an internal team workflow (not a demoed feature)
- Landing page, About/Methodology page, and a standalone Case Studies list page, in addition to the map
- Admin area: authenticated login, data refresh triggers, pipeline run health/log, case-study management, source-weight configuration
- Mobile-responsive design for all public-facing pages (admin area is desktop-only for V1)
- Written validation methodology and results (case-study evidence, to the extent measurable)

**Explicitly out of scope for V1 (deferred to later versions or Grand Vision — see 1.8):**
- Nationwide or multi-city coverage (V4)
- True forecasting of future hotspots, prices, or infrastructure needs (V5)
- Business-registration-based ground truth or validation
- Traffic data, social media trend data, and weather data as model inputs
- Public user accounts, saved areas/alerts, and custom report requests (V2)
- Paid tiers, payments, and API access (V3–V4)
- Dynamic/multi-resolution map grid (Grand Vision technical improvement)
- Real-time/continuously live data updates (admin-triggered refresh is sufficient for V1)

---

### 1.6 Success Criteria

- [ ] A visitor can explore the Addis Ababa map and view the composite growth score by grid cell without creating an account
- [ ] A visitor can move the time-slider and observe how the growth score for a given area changes over the available time range
- [ ] A visitor can click a grid cell and see the individual data signals, an AI-generated summary, and the score's recent trend
- [ ] A visitor can use natural-language search to filter the map
- [ ] At least 2–4 known, independently verifiable growth areas in Addis Ababa are marked on the map with supporting evidence and dates
- [ ] The composite growth score for at least one validated case study area rises before or alongside independent public confirmation of that growth
- [ ] The project team can trigger a data refresh and case-study update from the admin area without manual reprocessing
- [ ] The dashboard displays and functions correctly on mobile phones
- [ ] The methodology, validation approach, and limitations are clearly documented and defensible

---

### 1.7 Assumptions & Open Questions

| # | Assumption / Question | Owner | Resolved? |
|---|---|---|---|
| 1 | The MVP will detect current/recent growth signals only; it will not forecast future hotspots | Project Team | Yes |
| 2 | Addis Ababa is the sole geographic scope for V1; a second city is a V4 item, not a V1 commitment | Project Team | Yes |
| 3 | Business-registration data is not reliably accessible in bulk for Ethiopia at this time and will not be used as ground truth for V1 | Project Team | Yes |
| 4 | Validation relies on independently verifiable public evidence (tiered sourcing: official announcements, market reports, infrastructure coverage, local news) for 2–4 case studies, not statistical validation against a large dataset | Project Team | Yes |
| 5 | The four selected data sources (VIIRS, GHSL, Meta RWI, GDELT) provide sufficient signal for a credible composite growth score | Project Team | Pending — to be confirmed during methodology testing |
| 6 | OpenStreetMap data is used only as supplementary context, not a core model input, due to incomplete and uneven Ethiopia coverage | Project Team | Yes |
| 7 | V1 requires no public user authentication; an admin-only area is sufficient. Public accounts are a V2 item | Project Team | Yes |
| 8 | The illustrative customer segment for V1's narrative is development finance / NGO / government planning use, with a full multi-segment paid model deferred to V3+ | Project Team | Yes |
| 9 | Grid cell size is fixed (not dynamic) for V1, matched to Meta RWI's ~2.4km resolution; exact value TBD by the team | Project Team | Pending — exact cell size to be finalized |
| 10 | Spatial data is stored as plain lat/lng floats and a GeoJSON JSON column, without the PostGIS extension, for V1 | Project Team | Yes |
| 11 | Composite score weights (per data source) are configurable and versioned but not yet finalized in value | Project Team | Pending — to be confirmed during methodology testing |

---

### 1.8 Version Roadmap

The project is planned as a sequence of versions from the current MVP toward the full Grand Vision. **Only V1 is a committed build target for this release; V2–V5 are planning/presentation material**, to be built only if V1 is complete and time genuinely remains.

- **V1 — Detection Core (this release):** Everything in Section 1.5's in-scope list.
- **V2 — Identity & Requests:** Optional public accounts (map stays viewable without login); manual "request a custom report" flow; save/bookmark areas.
- **V3 — Monetization Layer:** Paid tier (priority reports, data export, saved-area alerts); payment integration; semi-automated, human-reviewed report generation.
- **V4 — API & Scale:** Read-only paid API access; second city added; institutional multi-area dashboard.
- **V5 — Forecasting & National Platform:** Requires a real ground-truth data partnership (not currently available) — nationwide coverage, full subscription/API/report business model, true forecasting.

---

**Next:** proceed to → [2. Functional & Non-Functional Requirements]
