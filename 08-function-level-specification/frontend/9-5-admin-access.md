## Project: Shadow Economy Map — Frontend Function-Level Spec: Admin Access
Files covered: `src/pages/admin/AdminLoginPage.tsx` · `src/store/adminAuth.store.ts` · `src/hooks/useAdminAuth.ts`

---

### src/store/adminAuth.store.ts (new — full block)

| Field | Detail |
|---|---|
| Shape | `{ token: string \| null; admin: { id: string; email: string } \| null; setAuth: (token: string, admin: { id: string; email: string }) => void; logout: () => void }` |
| Persistence | Zustand `persist` middleware, storage key `admin-auth-storage` (per frontend conventions 0.3) — `token` and `admin` survive a page refresh; nothing else in this project uses persisted client state |
| `setAuth` | Sets both `token` and `admin` in one call — never set independently, since a `token` without a matching `admin` (or vice versa) is an invalid intermediate state that `useAdminMe`/route guards could observe |
| `logout` | Clears both fields to `null`. Does **not** itself call `POST /admin/auth/logout` — the API call and the store clear are composed together only inside `useAdminLogout` below (per frontend spec 4.2), so nothing else in the codebase should call `logout()` directly and skip the API call |
| Edge cases | A stale/expired persisted token is not proactively cleared by the store itself — it is discovered lazily via `useAdminMe`'s 401 (which the axios interceptor turns into a forced logout + redirect, per frontend conventions 0.6) |
| Test file | `tests/store/adminAuth.store.test.ts` |

---

### src/hooks/useAdminAuth.ts (new — full block, per frontend spec 4.4)

#### useAdminLogin

| Field | Detail |
|---|---|
| Signature | `useAdminLogin(): UseMutationResult<AdminLoginResponse, AxiosError, { email: string; password: string }>` |
| Purpose | Wraps `POST /admin/auth/login`. |
| Side effects | `onSuccess` calls `useAdminAuthStore.setAuth(data.token, data.admin)` — no query invalidation needed, since no admin query can have run before a token exists |
| Edge cases | A `401 "Invalid email or password"` is a genuine mutation error (not a `parsedFilters`-style soft failure) — `AdminLoginPage` renders it as a form-level error banner, per frontend spec 4.5, since the client-side Zod schema cannot catch a valid-shaped-but-wrong credential pair |
| Test file | `tests/hooks/useAdminAuth.test.ts` |

#### useAdminLogout

| Field | Detail |
|---|---|
| Signature | `useAdminLogout(): UseMutationResult<void, AxiosError, void>` |
| Purpose | Wraps `POST /admin/auth/logout`. |
| Side effects | `onSettled` (not `onSuccess`) calls `useAdminAuthStore.logout()` — local session state is cleared even if the API call itself fails (e.g., the token had already expired server-side), per frontend spec 4.2, so the admin is never stuck unable to log out client-side because of a network error |
| Edge cases | Called from `DashboardLayout`'s "Logout" nav action (per frontend spec 9-1 §`src/components/layouts/DashboardLayout.tsx`) — after `onSettled`, the `ProtectedRoute` guard's next render redirects to `ROUTES.ADMIN_LOGIN` since `token` is now `null` |
| Test file | `tests/hooks/useAdminAuth.test.ts` |

#### useAdminMe

| Field | Detail |
|---|---|
| Signature | `useAdminMe(): UseQueryResult<AdminIdentity>` |
| Purpose | Wraps `GET /admin/auth/me` — confirms session validity on load and supplies the identity shown in the admin nav. |
| Query key | `['admin-me']`, `enabled: !!token` (does not fire before a token exists) |
| Side effects | None directly — a `401` response is left to the axios response interceptor (frontend conventions 0.6), which forces `useAdminAuthStore.logout()` and redirects to `ROUTES.ADMIN_LOGIN` |
| Edge cases | `retry: false` — a failed `/me` call means an invalid/expired token (per FR-17's inactivity expiry), so retrying with the same bad token is pointless; the interceptor, not query retry logic, owns recovery |
| Test file | `tests/hooks/useAdminAuth.test.ts` — includes the 401-forces-logout case |

---

### src/pages/admin/AdminLoginPage.tsx (new)

| Field | Detail |
|---|---|
| Purpose | Admin credential entry point (UC-11). |
| Route guard + layout | `PublicRoute` (redirects to `ROUTES.ADMIN_DASHBOARD` if already authenticated, per frontend conventions 0.1) + `AuthLayout` |
| Local state | React Hook Form state only — `email`, `password` |
| Behavior | 1. Client-side Zod validation mirrors the backend's `loginSchema` (email required/valid, password required min 8 chars, per frontend spec 4.5) — catches malformed input before any request is sent. 2. On submit, calls `useAdminLogin().mutate({ email, password })`. 3. On success, the `PublicRoute` guard's next render (driven by the store's now-populated `token`) redirects to `ROUTES.ADMIN_DASHBOARD` — the page itself does not `navigate()` imperatively. 4. On a `401` mutation error, renders "Invalid email or password" as a form-level banner above the fields, per FR-16 — never attributed to a specific field, matching the backend's intentionally generic message. |
| Components | none beyond form primitives from the template |

**States:** idle (form) · submitting (button disabled, spinner) · error (401 banner, form re-enabled) · success (redirect via guard, no local success state to render)

---

**Next:** proceed to → [9-6. Frontend: Data Pipeline Management]
