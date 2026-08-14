## Project: Shadow Economy Map — Frontend Test Documentation: Admin Access
**Links back to:** [8-5. Frontend Function-Level Spec: Admin Access], [9-1. Frontend Test Documentation: Shared/Config]

Per the Frontend Testing Guide: hooks mock `@/lib/axios` and the `adminAuth` store's `setAuth`/`clearAuth`; the page mocks the hooks. This is a High-Scrutiny module (Guide Section 12) — auth state, redirects, and the generic-401-error contract all repeat patterns the Guide's own worked eco-9.2.1 example already flags as easy to get subtly wrong (e.g. leaking field-specific errors on a deliberately generic backend message).

---

### 9.1 Test File Map

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| `src/hooks/useAdminAuth.ts` | `tests/hooks/useAdminAuth.test.ts` | Unit (`renderHook`, mocked `@/lib/axios` + mocked `adminAuth.store`/router) | ☐ |
| `src/pages/admin/AdminLoginPage.tsx` | `tests/pages/admin/AdminLoginPage.test.tsx` | Component (RTL, mocked `useAdminAuth`) | ☐ |

---

### 9.2 Test Case Detail — useAdminAuth.test.ts

#### useAdminLogin

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Calls POST /admin/auth/login with form values | mocked `api.post` resolves `{data: {token, admin: {id, email}}}` | `renderHook(() => useAdminLogin())`, `.mutate({email, password})` | `api.post` called with `('/admin/auth/login', {email, password})` |
| On success, persists auth state via the store | mock `useAdminAuthStore.setAuth` | `.mutate(...)`, await success | `setAuth` called with `(token, admin)` from the response |
| On success, navigates to the admin dashboard | mock `navigate` | `.mutate(...)`, await success | `navigate` called with `ROUTES.ADMIN_DASHBOARD` — there is no redirect-param concept for admin login (unlike a customer-facing login with `?redirect=`), since there is exactly one destination an admin ever lands on |
| Invalid credentials failure propagates without side effects | mocked `api.post` rejects with a 401 AxiosError | `.mutate(...)` | `result.current.isError === true`; `setAuth` NOT called; `navigate` NOT called |
| The 401's message is passed through unmodified | mocked `api.post` rejects with `{response: {status: 401, data: {message: 'Invalid email or password'}}}` | `.mutate(...)` | `result.current.error.response.data.message === 'Invalid email or password'` — the hook does not rewrite or re-map this text; mapping it to UI copy is the page's job, matching the pattern already established for `useLogin`/`useForgotPassword` in the Guide's own worked examples |

#### useAdminLogout

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Calls POST /admin/auth/logout | mocked `api.post` resolves | `renderHook(() => useAdminLogout())`, `.mutate()` | `api.post` called with `('/admin/auth/logout')` |
| On success, clears auth state via the store | mock `useAdminAuthStore.clearAuth` | `.mutate()`, await success | `clearAuth` called |
| On success, navigates to admin login | mock `navigate` | `.mutate()`, await success | `navigate` called with `ROUTES.ADMIN_LOGIN` |
| Clears local auth state even if the network call fails | mocked `api.post` rejects (e.g. token already expired server-side) | `.mutate()` | `clearAuth` is still called — logging out must succeed from the user's perspective even if the server-side call errors, since the whole point is to discard a token the client no longer wants to use; this is the one mutation in this module explicitly *not* gated on `onSuccess` for its state-clearing side effect |

#### useAdminMe

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Fetches the current admin's profile | mocked `api.get` resolves `{id, email, createdAt}` | `renderHook(() => useAdminMe())` | `api.get` called with `('/admin/auth/me')` |
| Query key has no params | inspect cache | render | cache entry keyed exactly `[QUERY_KEYS.ADMIN_ME]` |
| 401 (missing/invalid/expired token) surfaces as an error state | mocked `api.get` rejects with a 401 AxiosError | render, await | `result.current.isError === true` — note this 401 is also independently handled by the Axios interceptor itself (9-1), which clears storage and redirects; this hook-level test only confirms the hook's own state transitions to `isError`, not the interceptor's redirect behavior, which is `axios.test.ts`'s job |

---

### 9.3 Test Case Detail — AdminLoginPage.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders email and password fields | `vi.mock('@/hooks/useAdminAuth')` → mocked `useAdminLogin` idle state | render `<AdminLoginPage />` | email and password inputs present |
| Client-side validation blocks an invalid email format | mocked hook `mutate` | fill email `"not-an-email"`, submit | `mutate` NOT called; a field-level error is shown for email |
| Valid submit calls mutate with form values | mocked hook `mutate` | fill valid email/password, submit | `mutate` called with `{email, password}` |
| onError always sets a root-level error, never a field-specific one | mocked hook error state carrying `{response: {status: 401, data: {message: 'Invalid email or password'}}}` | trigger error handling | a root-level error message reads "Invalid email or password"; the email/password fields themselves show NO field-specific error styling — matches the backend's deliberately generic 401 (FR-16), same discipline as the Guide's `LoginPage` worked example |
| isSubmitting disables the button and shows a loading indicator | mocked hook state `isPending: true` | render | submit button disabled; loading indicator visible |
| Page itself does not navigate or set auth state on success | mocked hook simulating its own internal `onSuccess` (setAuth + navigate happen inside the hook, per 9.2 above) | submit and resolve success | no navigation call and no store mutation originates from the page component itself — confirms this responsibility is fully delegated to the hook, not duplicated |

---

### 9.4 Coverage Honesty Check (per PR Steward, at review time)

- [ ] The 401 generic-message case is asserted by checking the email/password fields do NOT carry error styling, not only that a root message is present — a page that (incorrectly) marks a field AND shows the root message would still pass a test that only checks for the root message, per the Guide's own explicit callout of this exact failure mode
- [ ] `useAdminLogout`'s "clears state even on network failure" case is a genuine, separate test from the success-path clear — not assumed to follow from the happy-path test alone
- [ ] `useAdminLogin`'s navigation-target assertion checks the specific `ROUTES.ADMIN_DASHBOARD` constant, not just that `navigate` was called with *some* string
- [ ] Page-level tests confirm navigation and store mutation do NOT originate from `AdminLoginPage` itself, isolating both responsibilities to the hook — consistent with 9-1's equivalent check for `DashboardLayout`'s logout button
- [ ] No test hardcodes the resolved `admin` object's shape inside the test's own JSX/assertions independent of what the mock actually returned — the rendered/passed data must be traced back to the mock, not duplicated by coincidence

---

### 9.5 Out of Scope for Automated Testing (and why)

- **Real session-expiry timing (FR-17's inactivity-based expiry)** — the frontend suite tests that a 401 is handled correctly when it occurs; the actual server-side timing of when a session becomes invalid is a backend concern (covered in the backend test doc), not something a frontend unit test simulates realistically.
- **Rate-limiting behavior on repeated failed login attempts** — this is enforced server-side (Doc 2's Security NFR); the frontend simply displays whatever error the backend returns, and this suite does not attempt to simulate a rate-limit-specific error response as a distinct case from any other 401/400.
- **Real browser autofill/password-manager interaction with the login form** — `userEvent.type` exercises the form's own validation and submit wiring; real browser autofill behavior is a manual QA concern.

---

**Next:** proceed to → [9-6. Frontend Test Documentation: Data Pipeline Management]
