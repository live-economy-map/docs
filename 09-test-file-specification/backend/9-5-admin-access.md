## Project: Shadow Economy Map — Backend Test Documentation: Admin Access
**Links back to:** [8-5. Function-Level Spec: Admin Access]

Per the standing rule in `7a-backend-file-structure.md`: test-first, test file mirrors `src/` exactly under `tests/`, never co-located. Vitest — `describe`/`it`/`expect`, mocks via `vi.fn()`/`vi.mock()`, `beforeEach(() => vi.clearAllMocks())`.

---

### 9.1 Test File Map

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| src/schemas/adminAuth.schema.ts | tests/schemas/adminAuth.schema.test.ts | Unit (schema parsing) | ☐ |
| src/services/adminAuth.service.ts | tests/services/adminAuth.service.test.ts | Unit (mocked Prisma + bcrypt) | ☐ |
| src/controllers/adminAuth.controller.ts | tests/controllers/adminAuth.controller.test.ts | Unit (mocked adminAuth.service via `vi.mock`) | ☐ |
| src/routes/adminAuth.routes.ts | tests/routes/adminAuth.routes.test.ts | Integration (supertest, mocked service layer + mocked `authMiddleware`) | ☐ |

---

### 9.2 Test Case Detail — adminAuth.schema.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| loginSchema rejects invalid email | — | parse `{ body: { email: "not-an-email", password: "somepassword" } }` | validation fails |
| loginSchema rejects short password | — | parse `{ body: { email: "admin@example.com", password: "short" } }` | validation fails (min 8 not met) |
| loginSchema accepts valid credentials shape | — | parse `{ body: { email: "admin@example.com", password: "validpassword123" } }` | passes |

`logout` and `getMe` have no schema (per Doc 8-5) — no corresponding schema test cases; both routes rely solely on `authMiddleware`-attached `req.user`.

---

### 9.2 Test Case Detail — adminAuth.service.test.ts

#### authenticateAdmin

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Valid credentials | mock `prisma.admin.findUnique` to return an admin row; mock `bcrypt.compare` → `true` | call `authenticateAdmin(email, password)` | resolves `{ token, admin: { id, email } }` |
| Email doesn't exist | mock `prisma.admin.findUnique` → `null` | call `authenticateAdmin(badEmail, anyPassword)` | throws `ApiError(401, "Invalid email or password")` |
| Wrong password | mock `findUnique` to return an admin row; mock `bcrypt.compare` → `false` | call `authenticateAdmin(email, wrongPassword)` | throws the exact same `ApiError(401, "Invalid email or password")` — assert identical status/message to the "email not found" case, per FR-16 and Doc 8-5's explicit instruction to test this |
| Timing — bcrypt.compare still runs when email not found | mock `findUnique` → `null`; spy on `bcrypt.compare` | call `authenticateAdmin(badEmail, anyPassword)` | assert `bcrypt.compare` was still called (against a dummy/constant hash) even though no real admin row was found — per Doc 8-5's timing-hardening note; a service that short-circuits before calling `bcrypt.compare` on a not-found email would fail this test even though the status/message assertion above would still pass, which is exactly why this needs its own case |
| Token issued with the correct admin id | mock `findUnique`/`bcrypt.compare` for a valid login; decode or spy on the JWT-signing call | call `authenticateAdmin(email, password)` | assert the signed token's payload (or the signing call's args) contains the admin's `id`, not e.g. the email alone or a placeholder |
| Password never included in the resolved result | mock a valid login | call `authenticateAdmin(email, password)` | assert `result.admin` has no `password`/hash field — only `id` and `email` |

#### invalidateSession

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Resolves without error for a valid token | — | call `invalidateSession(validToken)` | resolves `undefined`/void, does not throw |
| Behavior is documented, not assumed — *implementation-pending, per Doc 8-5* | — | — | **Open item:** whether this is a true no-op or writes a short-lived denylist entry is explicitly undecided in Doc 8-5 ("needs to be decided by whoever implements auth"). Do not write a test asserting denylist-write behavior until that decision is made — a premature test here would either be trivially vacuous (no-op case) or would need rewriting the moment the real decision is made. Once decided, add: if no-op, a case confirming no Prisma/cache write occurs; if denylist, a case confirming the entry's TTL matches the token's remaining lifetime. |

---

### 9.3 Test Case Detail — adminAuth.controller.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| login delegates correctly | `vi.mock` adminAuth.service; `authenticateAdmin` resolves `{token, admin}` | call controller with `req.body` | `authenticateAdmin` called with `req.body.email`, `req.body.password`; responds 200 via `SuccessResponse(200, "Login successful", {token, admin})` |
| login propagates 401 | mock `authenticateAdmin` to throw `ApiError(401, "Invalid email or password")` | call controller | error passed through via `asyncHandler`, not swallowed or rewritten |
| logout delegates correctly | mock `invalidateSession` resolves void | call controller with the request's token | `invalidateSession` called with the token; responds 200 with `SuccessResponse(200, "Logged out", {})` |
| getMe skips the service layer | set `req.user` directly (as `authMiddleware` would attach it) | call controller | responds 200 with `SuccessResponse(200, "OK", { id, email, createdAt })` derived from `req.user` alone; assert **no** `adminAuthService` function was called — per Doc 8-5's explicit note that this handler is a deliberate exception to the usual controller→service pattern |

---

### 9.3 Test Case Detail — adminAuth.routes.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| POST /login has no auth requirement | mock controller layer, no Authorization header | request `POST /admin/auth/login` with a valid body | 200, not 401 — login is necessarily public |
| POST /login validates body shape | mock controller layer | request `POST /admin/auth/login` with `{ email: "bad" }` (missing password, invalid email) | rejected by `validate(loginSchema)`, controller mock never called |
| POST /logout requires auth | no Authorization header | request `POST /admin/auth/logout` | 401, controller mock never invoked |
| GET /me requires auth | no Authorization header | request `GET /admin/auth/me` | 401, controller mock never invoked |
| POST /logout and GET /me succeed with a valid token | mock `authMiddleware` to pass through and attach `req.user`; mock controller layer | request each with a valid Authorization header | both reach their respective controller mocks, 200 |

---

### 9.4 Coverage Honesty Check (per PR Steward, at review time)

- [ ] The "email not found" and "wrong password" cases in `authenticateAdmin` assert the *identical* `ApiError` — this is the specific case Doc 8-5 calls out (per FR-16) as needing explicit coverage, and it's the easiest one to accidentally get subtly wrong (e.g., two similar-but-not-identical error messages)
- [ ] The timing-hardening case (`bcrypt.compare` still runs on a not-found email) is actually asserted via a spy on the call itself, not inferred from the error message alone — a service that returns the right error but skips the dummy-hash comparison would pass the message-only test while still leaking timing information
- [ ] `getMe`'s "skips the service layer" behavior is asserted by confirming no `adminAuthService` function was invoked, not just that the response shape looks right — a future refactor that accidentally reintroduces a redundant DB call would only be caught this way
- [ ] No test hardcodes a JWT string that isn't actually produced by the code under test (e.g., asserting `token === "mock-token"` without verifying the signing function was actually called with the right payload)

---

### 9.5 Out of Scope for Automated Testing (and why)

- **`invalidateSession`'s real behavior** — deliberately left thin above until the no-op-vs-denylist decision (Doc 8-5) is made; do not backfill a test that assumes one implementation over the other.
- **Real bcrypt hashing/comparison performance** — unit tests mock `bcrypt` entirely; real hash timing under load is a manual/perf concern, same precedent as the auth module in the reference project.
- **JWT expiry/session-inactivity timeout behavior (FR-17)** — unit tests can verify a token is issued and that `authMiddleware` rejects a missing/malformed token, but exercising real wall-clock expiry (inactivity timeout) needs either time-mocking (`vi.useFakeTimers()`, if adopted) or a manual/staging check; not assumed covered by the cases above without that explicit setup.
- **Rate limiting on repeated failed login attempts** (Doc 2's NFR) — covered structurally (is a rate-limiter middleware wired to the login route) but not load-tested for exact threshold numbers, matching the reference project's own precedent for rate-limiter testing.

---

**Next:** proceed to → [9-6. Backend Test Documentation: Data Pipeline Management]
