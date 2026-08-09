# Shadow Economy Map — Version Roadmap (V1 → V5)

## Purpose of This Document

This roadmap shows the staged path from the current MVP (V1) to the full Grand Vision. It exists so the project can be presented as "a concrete plan with a proven first stage," rather than a disconnected MVP and a vague wish list.

**Critical framing, stated plainly:** Only V1 is a committed build target. V2–V5 are planning and presentation material. This document is not a sprint plan — it is a roadmap to be shown, not a backlog to be executed in parallel with V1.

---

## Why a Roadmap Instead of "Just an MVP"

Every real product has a gap between its first version and its long-term vision — that gap is normal, not a flaw. The risk isn't that the gap exists; it's not being able to explain how you get from one side to the other. This document is that explanation.

It also directly answers a question any technical evaluator or investor will ask: **"What services would customers actually pay for, and when?"** Section "Monetization by Segment" below answers this concretely.

---

## Build Priority — Read This Before Anything Else

**V1 must be complete, polished, and validated before any V2 work begins.** This is a deliberate sequencing decision, not a soft suggestion:

- V1's core claim (that the composite growth score is a real, meaningful signal) is not yet proven at the time of writing. Case studies are not yet validated, the scoring pipeline is not yet built and tested.
- Building authentication, payments, or custom-report infrastructure (V2/V3) before V1's detection claim is solidly proven means building on an unproven foundation — if the scoring approach needs rework after testing, everything built on top of it becomes unstable too.
- Three to four case studies (V1's validation target) are enough to **prove the concept**, not enough to **responsibly back a paid product** that customers make real decisions from. Shipping monetization ahead of validation would be a bigger credibility risk than presenting a well-built free tool.
- Time and team realities: V1 was scoped specifically to fit within the available timeline for a software-engineering-strong team. Every additional version adds real, non-trivial build time (auth, payment integration, alerting, semi-automated reporting). Splitting focus risks a weaker V1 *and* an unfinished V2 — the worst possible outcome.

**The one allowed exception:** if V1 is fully complete (validated case studies, polished map, working AI features) with genuine spare time remaining, a single small, low-risk V2 item — such as optional accounts plus a manual "request a report" contact form (near-zero backend complexity) — may be added as a stretch goal. This is never a parallel track that risks V1's quality, and is not planned or assumed by default.

**Presentation strategy:** demo the completed version (V1, or V1 plus the one optional stretch item) live and working. Present this roadmap as the concrete plan for everything beyond it — this answers "is this just a student toy" without requiring unproven features to actually exist.

---

## V1 — Detection Core (This Release)

**Status: Committed build target.**

**What it is:** A public, account-free, growth-*detection* tool for Addis Ababa.

**Includes:**
- Interactive map with composite growth score per grid cell, time-slider, raw-layer toggle (VIIRS/GHSL/RWI)
- Click-to-inspect panel: composite score, raw signal breakdown, AI-generated plain-language summary, trend arrow, sparkline history, location label, freshness/completeness indicators
- Natural-language map search (AI-assisted query parsing)
- 2–4 validated case studies, sourced via tiered evidence (official announcements → market reports → infrastructure coverage → local news), each with dated evidence and, where available, before/after satellite imagery
- Landing page, About/Methodology page, standalone Case Studies list page
- Admin area: authenticated login, data pipeline refresh + health monitoring, case-study management, source-weight configuration
- Mobile-responsive public pages (admin is desktop-only)

**Explicitly not included:** public accounts, payments, API access, multi-city coverage, forecasting.

**Why this scope:** every inclusion/exclusion decision here follows the same rule — build only what can be honestly validated with currently accessible, free data, within the available timeline. See `1-problem-solution.md` Section 1.4 for the full reasoning behind each specific cut.

---

## V2 — Identity & Requests

**Status: Roadmap only — not a committed build.**

**What it adds:**
- Optional public user accounts. The map remains fully viewable without login — accounts add convenience, they don't gate the core value.
- A "request a custom report" flow: a logged-in user submits a specific question or area of interest; the project team fulfills it **manually**. This validates real demand before any automation is built.
- Save/bookmark specific areas of interest for return visits.

**Why this comes before monetization:** it proves people actually want a custom answer to a specific question — the thing V3 would later try to charge for — using the cheapest possible way to test that (a human doing the work manually, not an automated pipeline).

---

## V3 — Monetization Layer

**Status: Roadmap only — not a committed build.**

**What it adds:**
- A paid tier: priority turnaround on custom reports, data export, and saved-area alerts (notification when a bookmarked area's score changes meaningfully).
- Payment integration (e.g., Chapa, consistent with tooling already familiar to the broader team from other projects).
- Semi-automated report generation: AI-assisted drafting of a custom report, always human-reviewed before delivery — not a fully automated black box.

**Precondition:** V2 must have shown real demand (actual report requests from real users) before this is worth building — building payments for a feature nobody has asked for yet would be premature.

---

## V4 — API & Scale

**Status: Roadmap only — not a committed build.**

**What it adds:**
- Read-only API access, rate-limited, available on a paid tier.
- A second city added to prove the data pipeline and scoring methodology generalize beyond Addis Ababa specifically, not just work for one hand-tuned case.
- An institutional dashboard: multi-area tracking, usage history, suited to a bank or NGO account managing several regions of interest at once.

**Precondition:** V3 must have at least one real paying or piloting customer before scaling infrastructure and adding a second city — scaling before demand is proven is a common startup failure mode, avoided here deliberately.

---

## V5 — Forecasting & National Platform

**Status: Roadmap only — long-term, explicitly conditional.**

**What it adds:**
- True forecasting: predicting future growth, not just detecting current/past activity.
- Nationwide coverage.
- The full commercial platform: subscriptions, premium reports, and API access across all customer segments.

**Hard precondition — cannot be skipped:** forecasting requires a real ground-truth dataset (e.g., reliable business-registration data, or an equivalent historical outcome record) to train and validate against. This does not currently exist in accessible form for Ethiopia. Attempting forecasting before this precondition is met would produce an unprovable, potentially misleading claim — directly against the project's core credibility principle established from V1 onward. **V5 is not a matter of engineering effort alone; it depends on an external data-access breakthrough that has not yet happened.**

---

## Monetization by Segment (Full Grand Vision, V3+)

Nobody pays for the free public map — it exists to prove the signal is real. Paid value always takes the form of "run this on our specific question," not self-serve map access (until V4's API, which is itself paid).

| Segment | What they'd pay for | Earliest version this becomes available |
|---|---|---|
| NGOs / development finance orgs | Custom growth-ranking reports; assessment of a named region under consideration | V3 (manual, validated in V2) |
| Government planning offices | Infrastructure-gap analysis — rising-activity areas with low existing infrastructure investment | V3 |
| Real estate firms / investors | Early-signal alerts before an area's growth is widely known; due-diligence add-on reports for specific locations | V3–V4 (requires more track record/trust than NGO segment) |
| Banks | Custom risk/opportunity scoring for lending decisions in specific neighborhoods | V4 (highest trust bar, latest realistic entry point) |

**Why NGOs/development orgs are the realistic first customer, not banks or investors despite being listed first in the original project brief:** organizations of this kind already use directly comparable data — Meta's own Relative Wealth Index (one of V1's four core inputs) is used by real humanitarian and development organizations for exactly this kind of "where is need or opportunity concentrated" question. Banks and investors have higher trust bars and existing research infrastructure; they are a credible later-stage sell once V1's validation has matured into a longer track record.

---

## Summary Table

| Version | Core Addition | Status |
|---|---|---|
| V1 | Public detection map, AI features, validated case studies, admin tools | **Committed — this release** |
| V2 | Optional accounts, manual custom-report requests | Roadmap only |
| V3 | Payments, paid tier, semi-automated reports | Roadmap only |
| V4 | API access, second city, institutional dashboard | Roadmap only |
| V5 | Forecasting, nationwide coverage, full platform | Roadmap only — blocked on external data access |
