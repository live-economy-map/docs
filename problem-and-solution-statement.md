## 1. Problem & Solution Statement

**Project:** Shadow Economy Map

**Owners:**

| No. | Name | Id |
|---|---|---|
| 1 | | |
| 2 | | |
| 3 | | |
| 4 | | |
| 5 | | |

**Status:** Draft

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
| Admin | The project/data team maintaining the platform | Manage data pipeline runs, refresh source data, and monitor the composite scoring outputs, without manual reprocessing for each update |

*(Future actors — not part of the MVP): Bank/investment analyst, real estate market researcher, government infrastructure planner.*

---

### 1.3 Proposed Solution

#### Grand Vision

A geospatial growth-signal platform that detects and, eventually, predicts emerging economic activity across Ethiopia, using satellite, mobility, and public data to surface growth before it appears in official statistics. At full scale, this platform would cover the whole country, integrate an evolving mix of data sources (satellite imagery, night-time lights, traffic data, business registrations, social trends, and more as they become reliably accessible), forecast future hotspots rather than only detecting current activity, and serve multiple paying customer segments — banks, investors, real estate companies, government agencies, and NGOs — through a live subscription platform with API access and custom reports.

This is the long-term direction the project is working toward. It is not the version being built or validated in this release.

#### MVP (This Release)

A geospatial growth-*detection* tool for Addis Ababa, built entirely on free, high-frequency, publicly accessible data.

Anyone can explore the interactive map without an account, browsing the composite growth score across grid cells and moving through time via a time-slider to see how signals have changed. Clicking a grid cell reveals the underlying signal breakdown — night-light trend, built-up area change, relative wealth estimate, and related news activity — rather than a single opaque number. A small set of known, independently-verifiable growth areas in Addis Ababa are marked directly on the map as case studies, each annotated with the date the growth score rose and the date that growth was independently confirmed (e.g., by news coverage or a published report), forming the project's core validation evidence. An admin-only area allows the project team to trigger data pipeline refreshes and monitor source data health.

The MVP does not forecast future hotspots or prices — it detects and displays current and recent activity only, and is scoped to a single city rather than the whole country.

---

### 1.4 Why This Approach (vs. alternatives considered)

**Prediction vs. detection:** A full predictive model (forecasting future hotspots, six months or more out) was considered, matching the original project concept. It was set aside for this release because predictive claims require historical ground truth to train and validate against, and reliable business-registration or economic ground-truth data is not currently accessible in bulk for Ethiopia. Building a "prediction" feature without the ability to validate it would produce an unprovable and potentially misleading claim. Detection of current, ongoing growth — validated against independently verifiable public evidence — was chosen instead as the honestly provable version of the same idea. Forecasting remains a Grand Vision goal once better ground-truth data becomes available.

**National vs. single-city scope:** Nationwide coverage was considered, matching the original vision. It was set aside for this release because data quality and availability vary significantly across regions, and committing to national scope up front risked shallow, unvalidated coverage everywhere rather than a rigorously proven result in one place. Addis Ababa was chosen as the initial scope because it has the strongest data density across all selected sources — accepted trade-off being that generalization to other regions is deferred to future work.

**Broad data mix vs. four core sources:** The original concept proposed seven or more data sources (including traffic, social media, weather, and business registrations). Several of these were set aside for this release: traffic and business-registration data are not reliably accessible in Ethiopia at this time, and social media and weather data provide weak or noisy signal relative to the engineering effort required. Four sources — VIIRS night-time lights, GHSL built-up area/population data, Meta's Relative Wealth Index, and GDELT news signals — were chosen because each is free, well-documented, and already purpose-built for exactly this kind of economic-proxy use case, reducing engineering risk for a team whose core strength is software engineering rather than novel ML research.

**Multi-segment platform vs. one illustrative customer:** The original plan listed five customer segments with no clear entry point. For this release, one illustrative customer angle — development finance / NGO / government planning use — was chosen to make the "who would use this" case concrete and credible, since organizations of this kind already use comparable data (e.g., Meta's Relative Wealth Index is used by real development organizations). A full multi-segment commercial platform remains part of the Grand Vision.

---

### 1.5 Scope

**In scope for this project (MVP):**
- Interactive map of Addis Ababa showing a composite growth score per grid cell, browsable without an account
- Time-slider to view how the growth score has changed over time
- Click-to-inspect view showing the individual signal breakdown per grid cell (night-lights, built-up area change, relative wealth estimate, news signal)
- Data pipeline integrating four sources: VIIRS night-time lights, GHSL built-up area/population, Meta Relative Wealth Index, GDELT
- A small set of marked, annotated case-study locations demonstrating validation against independently verifiable public evidence
- Admin area for the project team to trigger data refreshes and monitor pipeline/data health
- Mobile-responsive dashboard design
- Written validation methodology and results (accuracy/correlation against known case studies, to the extent measurable)

**Explicitly out of scope (for now):**
- Nationwide or multi-city coverage
- True forecasting of future hotspots, prices, or infrastructure needs
- Business-registration-based ground truth or validation
- Traffic data, social media trend data, and weather data as model inputs
- Multi-segment commercial platform (subscriptions, premium reports, API access for paying customers)
- User accounts, authentication, or role-based access beyond a single project-team admin area
- Real-time/continuously live data updates (periodic manual or scheduled refresh is sufficient for this release)

---

### 1.6 Success Criteria

- [ ] A visitor can explore the Addis Ababa map and view the composite growth score by grid cell without creating an account
- [ ] A visitor can move the time-slider and observe how the growth score for a given area changes over the available time range
- [ ] A visitor can click a grid cell and see the individual data signals contributing to its composite score
- [ ] At least 2–3 known, independently verifiable growth areas in Addis Ababa are marked on the map with supporting evidence (e.g., news or published reports) and dates
- [ ] The composite growth score for at least one validated case study area rises before or alongside independent public confirmation of that growth
- [ ] The project team can trigger a data refresh from the admin area without manual reprocessing of each individual data source
- [ ] The dashboard displays and functions correctly on mobile phones
- [ ] The methodology and its limitations (single-city scope, no registry-based ground truth, coarse RWI resolution, detection rather than forecasting) are clearly documented and defensible

---

### 1.7 Assumptions & Open Questions

| # | Assumption / Question | Owner | Resolved? |
|---|---|---|---|
| 1 | The MVP will detect current/recent growth signals only; it will not forecast future hotspots | Project Team | Yes |
| 2 | Addis Ababa is the sole geographic scope for this release; a second city is a stretch goal only, not a commitment | Project Team | Yes |
| 3 | Business-registration data is not reliably accessible in bulk for Ethiopia at this time and will not be used as ground truth for this release | Project Team | Yes |
| 4 | Validation will rely on independently verifiable public evidence (news, published reports) for a small number of case-study areas, rather than statistical validation against a large ground-truth dataset | Project Team | Yes |
| 5 | The four selected data sources (VIIRS, GHSL, Meta RWI, GDELT) provide sufficient signal for a credible composite growth score | Project Team | Pending — to be confirmed during methodology testing |
| 6 | OpenStreetMap data will be used only as supplementary context, not as a core model input, due to incomplete and uneven coverage across Ethiopia | Project Team | Yes |
| 7 | The MVP does not require user authentication for the public-facing map; an admin-only area is sufficient for the project team's own data management needs | Project Team | Yes |
| 8 | The illustrative customer segment for this release is development finance / NGO / government planning use, not a fully commercial multi-segment product | Project Team | Yes |

---

**Next:** proceed to → [2. Functional & Non-Functional Requirements]
