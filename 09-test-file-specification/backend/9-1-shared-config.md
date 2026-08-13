## Project: Shadow Economy Map — Backend Test Documentation: Shared / Config
**Links back to:** [7a. Backend Folder & File Structure], [8-1. Function-Level Spec: Shared / Config]

Per the standing rule in `7a-backend-file-structure.md` ("every service file gets a mirrored test file under `tests/`, per template convention"): test-first, test file mirrors `src/` exactly under `tests/`, never co-located. Test runner: Vitest — `describe`/`it`/`expect`, mocks via `vi.fn()` / `vi.mock()`, `beforeEach(() => vi.clearAllMocks())` to reset mock state between cases.

**Scope note:** this module covers files not owned by a single feature. Most of Doc 8-1's file list is explicitly excluded from automated testing (see 9.1 below) — `prisma/schema.prisma`, `prisma/seed.ts`, `src/app.ts`, `package.json`, and `.env.example` are config/declarative artifacts, not logic. Only `src/config/env.ts` (modified) and `src/utils/aiClient.ts` (new) contain testable behavior.

---

### 9.1 Test File Map

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| src/config/env.ts | tests/config/env.test.ts | Unit (Zod schema / fail-fast validation) | ☐ |
| src/utils/aiClient.ts | tests/utils/aiClient.test.ts | Unit (mocked LLM SDK client) | ☐ |
| prisma/seed.ts | — | Not required (see 9.5) | — |
| src/app.ts | — | Not required (see 9.5) | — |

**Assumed interface for `aiClient.ts` (flagged, not yet fixed elsewhere in the docs):** Doc 8-1 specifies only that this file "wraps the LLM API" for three callers — `map.service.ts`'s `generateCellSummary`/`parseNaturalLanguageQuery`, and `adminCaseStudies.service.ts`'s `searchCaseStudyCandidates`. Two of these need free-text output (a cell summary) and two need structured/parseable output (query filters, discovery candidates). This doc assumes `aiClient.ts` exposes two generic functions — `generateText(prompt: string): Promise<string>` and `generateStructuredOutput<T>(prompt: string, schemaHint: unknown): Promise<T | null>` — rather than one function per caller, so the wrapper stays caller-agnostic. **This should be confirmed with whoever implements `aiClient.ts`** before these test cases are finalized; if the real interface differs, the case list below still applies conceptually (text-generation path vs. structured-output path) but the function names/signatures will need updating.

---

### 9.2 Test Case Detail — env.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| All required vars present — loads successfully | set all of `DATABASE_URL`, `JWT_SECRET`, `GOOGLE_EARTH_ENGINE_CREDENTIALS`, `GDELT_API_KEY`, `LLM_API_KEY`, `ADMIN_SEED_EMAIL`, `ADMIN_SEED_PASSWORD` on `process.env` | import/parse the env schema | resolves without throwing; parsed object exposes all values, none `undefined` |
| Missing `GOOGLE_EARTH_ENGINE_CREDENTIALS` — fails fast | set every var except this one | import/parse the env schema | throws / exits, per the fail-fast philosophy already established for `JWT_SECRET`/`DATABASE_URL` — this is not a silently-optional var |
| Missing `LLM_API_KEY` — fails fast | set every var except this one | import/parse the env schema | throws / exits — the AI features (cell summary, NL search, discovery) have no degraded-but-working mode without it, so this must be a hard startup failure, not a runtime surprise |
| Missing `ADMIN_SEED_EMAIL` or `ADMIN_SEED_PASSWORD` | set every var except one of these two | import/parse the env schema | throws / exits — matches `prisma/seed.ts`'s own fail-fast behavior (Doc 8-1), asserting the *schema* also refuses to start, not just the seed script in isolation |
| `GDELT_API_KEY` requirement — *pending confirmation* | set every var except `GDELT_API_KEY` | import/parse the env schema | **Open item, do not implement until confirmed:** Doc 8-1 notes some GDELT endpoints are keyless ("confirm against GDELT's actual API docs during implementation"). If the chosen endpoint turns out to be keyless, this var should be optional and this test case should assert it does *not* fail fast; if a key is required, this case merges with the fail-fast cases above. Whoever implements `gdelt.client.ts` (Doc 8-6) should settle this first — this row exists so the ambiguity isn't silently guessed at in the test suite |
| Existing vars still validated (no regression) | omit `JWT_SECRET` only, all project-specific vars present | import/parse the env schema | still throws / exits — confirms the new required vars were *added* to the existing schema, not swapped in over it |

---

### 9.2 Test Case Detail — aiClient.test.ts

#### generateText

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Successful generation | mock the underlying LLM SDK client to resolve a text response | call `generateText(prompt)` | resolves the plain string content, not the raw SDK response envelope |
| Empty prompt | — | call `generateText("")` | rejects/throws before making an SDK call — an empty prompt is a caller bug, not a valid "summarize nothing" request; asserts the SDK mock was never invoked |
| SDK times out | mock the SDK client to reject with a timeout error | call `generateText(prompt)` | rejects with a normalized error the caller can distinguish from a malformed-output case (see `map.service.ts`'s "AI summary unavailable" vs. `map.service.ts`'s "search temporarily unavailable" distinction, per API spec 1.2 and 5.2's parallel case) — this wrapper should not leak the raw SDK error shape to callers |
| SDK returns an empty/whitespace-only response | mock the SDK client to resolve `""` or `"   "` | call `generateText(prompt)` | treated as a failure, not a successful-but-empty summary — callers (`generateCellSummary`) must not display a blank AI section as if it were a real (if terse) summary |

#### generateStructuredOutput

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Successful parse | mock the SDK to resolve a JSON-shaped string matching the expected schema | call `generateStructuredOutput(prompt, schemaHint)` | resolves the parsed object, not the raw string |
| SDK returns malformed/non-JSON text | mock the SDK to resolve prose instead of JSON (a known LLM failure mode) | call `generateStructuredOutput(prompt, schemaHint)` | resolves `null`, does **not** throw — this is the case that backs `map.service.ts`'s "couldn't confidently parse the query, leave the map unchanged" alternate flow (UC-10) and `adminCaseStudies.service.ts`'s discovery no-results case; a thrown error here would incorrectly surface as a 400/500 instead of the documented graceful no-op |
| SDK returns JSON that doesn't match the expected shape (e.g., missing required keys) | mock the SDK to resolve valid JSON with the wrong keys | call `generateStructuredOutput(prompt, schemaHint)` | resolves `null` — schema-shape mismatch is treated identically to non-JSON output, both are "couldn't get usable structured data," per the same graceful-degradation principle above |
| SDK is unreachable/service down | mock the SDK client to reject with a connection error | call `generateStructuredOutput(prompt, schemaHint)` | rejects (does **not** resolve `null`) — this is the distinct "service temporarily unavailable" case (API spec 1.2's `POST /map/search` 400 "Search is temporarily unavailable" and 6.2's `POST /admin/case-studies/discover` 400 "Discovery search is temporarily unavailable"), which callers must be able to tell apart from a clean "couldn't understand/find anything" `null` result — conflating the two would turn a genuine outage into a silent no-op |

---

### 9.4 Coverage Honesty Check (per PR Steward, at review time)

Use this checklist, don't just trust the green checkmark:

- [ ] `env.ts` tests actually reset `process.env` between cases (e.g., via `vi.stubEnv`/manual save-restore in `beforeEach`/`afterEach`) — a test order dependency here would silently mask a missing-var case
- [ ] The "malformed JSON" and "service unreachable" cases for `generateStructuredOutput` are asserted as genuinely different outcomes (`null` vs. thrown error) — not both collapsed into "it didn't work," which would let a caller mix up "nothing found" with "AI is down"
- [ ] No test is asserting against a hardcoded string that isn't actually derived from the mocked SDK response (e.g., a `generateText` test that would pass even if the wrapper ignored the mock and returned a fixture string)
- [ ] The `GDELT_API_KEY` case (flagged pending above) is not implemented as a guess — either it's confirmed and the test reflects a real decision, or it's left as a documented open item, not silently resolved one way in the test suite without a matching decision recorded in Doc 8-6

---

### 9.5 Out of Scope for Automated Testing (and why)

- **`prisma/seed.ts`** — excluded from the mirrored-test-file rule per Doc 8-1's own note ("matching the e-commerce template's own precedent, which also didn't test `seed.ts`"). Idempotency (safe re-running via `upsert`) is verified manually against a real dev database, not mocked.
- **`src/app.ts`** — no change from the base template (Doc 8-1: "requires zero deviation"), so no new test surface is introduced here; the template's own middleware-order tests (if any) already cover it.
- **Real Google Earth Engine / GDELT / LLM API integration** — `aiClient.ts` and the data source clients are unit-tested against a mocked SDK only; actual network behavior, auth against live credentials, and real response-shape drift from these third-party APIs need a manual or separately-tracked integration pass, not this unit suite.
- **`config/env.ts` behavior under a real `.env` file (dotenv loading order, etc.)** — unit tests exercise the Zod schema directly against a stubbed `process.env`; whether the actual `.env`/`.env.example` file loads correctly at real startup is a manual boot-up check.

---

**Next:** proceed to → [9-2. Backend Test Documentation: Growth Map & Exploration]
