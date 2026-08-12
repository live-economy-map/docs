## Project: Shadow Economy Map — Backend Function-Level Spec: Shared / Config
Covers every backend file not owned by a single feature — schema, seed, app entry point, package/env config. Each feature (Growth Map, Case Studies, Site Content, Admin Access, Data Pipeline, Case Study Curation) has its own function-level spec file covering its schema/service/controller/route files.

---

### prisma/schema.prisma (modify)

Add every model and enum below. Full field-level detail already specified in Doc 4 §4.2 (as revised: `ScoreWeightConfig`/`SourceWeight` split) — this section restates only the enum values explicitly, since they're a common guessing point, and confirms no deltas exist beyond what Doc 4 already defines.

**Models to add:** Admin, DataSource, PipelineRun, GridCell, SignalValue, ScoreWeightConfig, SourceWeight, CompositeScoreSnapshot, CaseStudy — exact fields per Doc 4 §4.2, unchanged.

**Enums — exact values, no others:**
```prisma
enum DataSourceKey {
  VIIRS
  GHSL
  RWI
  GDELT
}

enum RunStatus {
  RUNNING
  SUCCESS
  FAILED
}

enum EvidenceTier {
  OFFICIAL
  MARKET_REPORT
  INFRASTRUCTURE
  LOCAL_NEWS
}
```

No `User` model, no `Role` enum — confirmed standing decision (Doc 4): V1 has exactly one authenticated entity, `Admin`, with no role field and no second role to distinguish.

---

### prisma/seed.ts (new)

| Field | Detail |
|---|---|
| Purpose | Bootstraps the single admin account, and seeds the four fixed `DataSource` rows — there is no admin-registration endpoint (per scope), so this is the only way an `Admin` ever comes to exist. `DataSource` rows must exist before any `PipelineRun` or `SignalValue` can reference them. |
| Reads from env | `ADMIN_SEED_EMAIL`, `ADMIN_SEED_PASSWORD` (new vars — see `.env.example` below) |
| Behavior | 1. Read `ADMIN_SEED_EMAIL`/`ADMIN_SEED_PASSWORD` from env. 2. Hash the password with the same `BCRYPT_SALT_ROUNDS` used everywhere else (import from `config/env.ts`, don't hardcode a separate value). 3. `prisma.admin.upsert()` keyed on `email` — upsert, not create, so re-running the seed doesn't crash on a duplicate-email error. 4. `prisma.dataSource.upsert()` for each of the four sources (`VIIRS`, `GHSL`, `RWI`, `GDELT`), keyed on `key`, setting `name`/`description` — also upsert, so re-seeding is safe and idempotent. |
| Throws | If `ADMIN_SEED_EMAIL`/`ADMIN_SEED_PASSWORD` are missing from env, exit the process with a clear error (fail-fast, matching `config/env.ts`'s philosophy) — don't silently skip seeding. |
| Edge cases | Re-running the seed after the admin's password was already changed via normal means — the upsert would overwrite the live password back to the seed value. Acceptable for this project (single-owner, low-stakes dev/practice project), but worth a code comment warning future maintainers. Re-running the seed after a `DataSource.description` was edited by an admin (not currently possible in V1 — no endpoint edits `DataSource` — but worth the same comment in case one is added later). |

Test file: not required — seed scripts are typically excluded from the mirrored-test-file rule (matching the e-commerce template's own precedent, which also didn't test `seed.ts`).

---

### src/app.ts (no change)

**Explicit statement, not an omission:** unlike the e-commerce template's project (which needed raw-body capture for a payment-provider webhook signature), **V1 has no webhooks** — no payment provider, no external service posting callbacks into our API. All four data source pulls (VIIRS, GHSL, RWI, GDELT) are outbound calls our own pipeline initiates and waits on; nothing calls back into us. `app.ts`'s middleware order therefore requires **zero deviation** from the base template — global `express.json()` applies to every route without exception.

If a future version introduces an inbound webhook (e.g., a payment provider in V3), this section would need the same kind of raw-body carve-out documented here as a new, explicit deviation at that time — not assumed now.

---

### package.json (modify)

| Dependency | Type | Purpose |
|---|---|---|
| `@google-cloud/earthengine` (or equivalent GEE client) | dependency | Outbound calls to Google Earth Engine for VIIRS and GHSL pulls, from `src/utils/dataSourceClients/viirs.client.ts` and `ghsl.client.ts` |
| An LLM SDK (e.g. `@anthropic-ai/sdk`) | dependency | Powers `src/utils/aiClient.ts` — cell summaries, natural-language query parsing, case-study discovery |
| A lightweight geometry lib (e.g. `@turf/turf`) | dependency, optional | Only if any point-in-polygon or grid-generation logic needs it server-side — not required for reading/writing `GridCell.boundaryGeoJson` as plain JSON, per Doc 4's no-PostGIS decision |

No `axios` addition needed beyond what the base template may already include — outbound calls to GDELT's REST API can use the template's existing HTTP client setup if one exists, or a minimal fetch wrapper; confirm against the actual template scaffold before adding a redundant dependency.

No changes to `scripts` beyond an optional convenience entry: `"prisma:seed": "tsx prisma/seed.ts"`.

---

### .env.example (modify)

| Variable | Purpose |
|---|---|
| `GOOGLE_EARTH_ENGINE_CREDENTIALS` | Service account credentials (JSON, or path to it) for `viirs.client.ts`/`ghsl.client.ts` |
| `GDELT_API_KEY` | If GDELT's endpoint used requires one — confirm against GDELT's actual API docs during implementation; some GDELT endpoints are keyless |
| `LLM_API_KEY` | Powers `aiClient.ts` |
| `ADMIN_SEED_EMAIL`, `ADMIN_SEED_PASSWORD` | Consumed only by `prisma/seed.ts` — never read anywhere else in the app |

`config/env.ts`'s Zod schema (per the base template's fail-fast pattern) must be extended to validate all of the above as required strings, exactly like the existing `JWT_SECRET`/`DATABASE_URL` entries — this is a small but necessary modification to `src/config/env.ts` itself, not just `.env.example`.

---

**Next:** proceed to → [8-2. Backend: Growth Map & Exploration]
