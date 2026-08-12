## Project: Shadow Economy Map — Backend Function-Level Spec: Admin Access
Files covered: `src/schemas/adminAuth.schema.ts` · `src/services/adminAuth.service.ts` · `src/controllers/adminAuth.controller.ts` · `src/routes/adminAuth.routes.ts`

---

### src/schemas/adminAuth.schema.ts (new)

| Schema | Shape |
|---|---|
| loginSchema | `z.object({ body: z.object({ email: z.string().email(), password: z.string().min(8) }) })` |

`logout` and `getMe` have no schema — no body/params/query, only the `authMiddleware`-attached `req.user`.

---

### src/services/adminAuth.service.ts (new)

#### authenticateAdmin

| Field | Detail |
|---|---|
| Signature | `authenticateAdmin(email: string, password: string): Promise<{ token: string; admin: { id: string; email: string } }>` |
| Purpose | Validates credentials, issues a JWT. |
| Inputs | `email`, `password` |
| Output | Token + admin identity |
| Throws | `ApiError(401, "Invalid email or password")` — email not found, or password mismatch, identical message either way (per FR-16 — never reveals which field was wrong) |
| Side effects | Read-only lookup, then a `bcrypt.compare`; no DB write |
| Edge cases | Timing — `bcrypt.compare` should still run (against a dummy hash) even when the email isn't found, so response time doesn't leak whether the email exists. This is a standard hardening detail worth calling out explicitly rather than leaving to chance during implementation. |

Test file: `tests/services/adminAuth.service.test.ts` — explicitly covers both "email not found" and "wrong password" returning the identical message.

#### invalidateSession

| Field | Detail |
|---|---|
| Signature | `invalidateSession(token: string): Promise<void>` |
| Purpose | Server-side portion of logout. |
| Inputs | The current request's raw token |
| Output | None |
| Throws | None |
| Side effects | **Open implementation detail, flagged in API spec 4.2:** since auth is stateless JWT with no session table in Doc 4's schema, this function's actual behavior (no-op vs. short-lived denylist entry) needs to be decided by whoever implements auth. If a denylist is chosen, it needs its own lightweight store (e.g., an in-memory cache with TTL matching the token's remaining lifetime, not a new Prisma model — a full DB table for this would be disproportionate for a single-admin system). |
| Edge cases | If implemented as a true no-op, `logout` is effectively "the client discards the token" — still a valid, honest V1 choice, just needs the API spec's message to stay accurate ("Logged out") regardless of which implementation is picked. |

Test file: `tests/services/adminAuth.service.test.ts`

---

### src/controllers/adminAuth.controller.ts (new)

| Handler | Calls | Response |
|---|---|---|
| login | `adminAuthService.authenticateAdmin(req.body.email, req.body.password)` | 200, `SuccessResponse(200, "Login successful", { token, admin })` |
| logout | `adminAuthService.invalidateSession(req.token)` | 200, `SuccessResponse(200, "Logged out", {})` |
| getMe | returns `req.user` directly (already attached by `authMiddleware`, no service call needed) | 200, `SuccessResponse(200, "OK", { id, email, createdAt })` |

`getMe` is the one handler in this spec collection that skips the service layer entirely — `authMiddleware` has already fetched and attached the admin record, so re-querying it in a service function would be a redundant DB call for no benefit. Worth noting as a deliberate exception to the usual controller→service pattern, not an inconsistency.

---

### src/routes/adminAuth.routes.ts (new)

| Method | Path | Middleware chain | Handler |
|---|---|---|---|
| POST | /login | `validate(loginSchema)` | login |
| POST | /logout | `authMiddleware` | logout |
| GET | /me | `authMiddleware` | getMe |

Mounted at `/admin/auth`. `authMiddleware` on `logout` and `me` only — `login` is necessarily public (can't require a token to obtain one).

---

**Next:** proceed to → [8-6. Backend: Data Pipeline Management]
