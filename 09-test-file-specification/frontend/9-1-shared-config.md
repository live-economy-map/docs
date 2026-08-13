## Project: Shadow Economy Map — Frontend Test Documentation: Shared / Config
**Links back to:** [0. Frontend Conventions], [7b. Frontend File Structure], [9-1. Frontend Function-Level Spec: Shared/Config]

Covers every frontend file not owned by a single feature — types, constants, routing shell, the Axios instance, the one Zustand store, and the two non-feature-specific layouts. Each feature (Growth Map, Case Studies, Site Content, Admin Access, Data Pipeline, Case Study Curation) has its own test documentation file covering its hooks/pages/components.

Per the **Frontend Testing Guide**: test-first, test file mirrors `src/` exactly under `tests/`, never co-located. Test runner: Vitest — `describe`/`it`/`expect`, mocks via `vi.fn()`/`vi.mock()`, `beforeEach(() => vi.mocked(fn).mockReset())`. Hooks/guards are tested with `renderHook`/`render` wrapped in a fresh `QueryClientProvider` and/or a scoped `createMemoryRouter` via the shared test utilities (Section 5 of the Guide) — never an ad hoc provider or the real app router. Components are tested with `render`/`screen`/`userEvent`.

---

### 9.0 Test Infrastructure (build before anything in 9.1–9.7)

Per Guide Section 5, these are infrastructure, not feature tests — they don't mirror a `src/` file and have no test file of their own, but every other test file in this project depends on them existing first.

| File | Purpose | Notes for this project |
|---|---|---|
| `tests/setup.ts` | Global setup — `@testing-library/jest-dom/vitest` matchers, `cleanup()` in `afterEach`, imports `resetStores.ts` so store resets run globally | Standard, per Guide Section 3 |
| `tests/utils/renderWithProviders.tsx` | Shared render wrapper — fresh `QueryClient` per test (`retry: false, gcTime: 0`), `QueryClientProvider` | Standard, per Guide Section 5.1 — no router wrapping baked in here, since public pages need no router context beyond what `createTestRouter` supplies per-test |
| `tests/utils/createTestRouter.tsx` | `renderRoutes(routes, initialEntries)` — builds a scoped `createMemoryRouter` and renders it inside `RouterProvider` | Per Guide Section 5.2. **This project's real `src/routes/index.tsx` now builds its router via `createBrowserRouter` (data-router API, not the classic `BrowserRouter`/`Routes`/`Route` tree)** — `createTestRouter` already uses the matching data-router test API (`createMemoryRouter`), so no divergence exists between how the real app routes and how tests route |
| `tests/utils/resetStores.ts` | Calls `resetAdminAuthStore()` in a global `afterEach` | Registers the **one** Zustand store this project has (Section 0.3 of the frontend conventions) — there is no second store to add here, unlike a project with separate cart/theme/etc. stores |

**Project-specific testability note:** for `tests/routes/index.test.tsx` (9.2 below) to build a scoped `createMemoryRouter` fed the *actual* route tree rather than a hand-copied duplicate of it, `src/routes/index.tsx` must export its route-object array as a named export (e.g. `export const routeObjects: RouteObject[] = [...]`) separately from the `router` instance (`export const router = createBrowserRouter(routeObjects)`) and the default-exported `<RouterProvider>` component. This mirrors the same "expose for testability" carve-out the Guide already makes for the Axios interceptor (Section 6.6/14.6) — a small, deliberate, non-hacky change to the source file, not a test-only workaround.

---

### 9.1 Test File Map

| Source file | Test file | Test type | Written before code? |
|---|---|---|---|
| `src/store/adminAuth.store.ts` | `tests/store/adminAuth.store.test.ts` | Unit (store, no render) | ☐ |
| `src/lib/axios.ts` | `tests/lib/axios.test.ts` | Unit — special case, mocks `axios` itself, not `@/lib/axios` (Guide 6.6) | ☐ |
| `src/routes/ProtectedRoute.tsx` | `tests/routes/ProtectedRoute.test.tsx` | Component (RTL, `createMemoryRouter`, store set directly) | ☐ |
| `src/routes/PublicRoute.tsx` | `tests/routes/PublicRoute.test.tsx` | Component (RTL, `createMemoryRouter`, store set directly) | ☐ |
| `src/components/layouts/PublicLayout.tsx` | `tests/components/layouts/PublicLayout.test.tsx` | Component (RTL, `createMemoryRouter` with a child route) | ☐ |
| `src/components/layouts/DashboardLayout.tsx` | `tests/components/layouts/DashboardLayout.test.tsx` | Component (RTL, mocked `useAdminLogout`) | ☐ |
| `src/routes/index.tsx` | `tests/routes/index.test.tsx` | Integration (RTL, real guards + layouts, `createMemoryRouter` fed `routeObjects`) | ☐ |

`src/types/index.ts` and `src/constants/index.ts` have **no dedicated test file** — see 9.5.

---

### 9.2 Test Case Detail — adminAuth.store.test.ts

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Initial state has no token/admin | — | `useAdminAuthStore.getState()` | `token: null`, `admin: null` |
| setAuth stores token and admin identity | — | `useAdminAuthStore.getState().setAuth('jwt-abc', {id, email})` | `getState().token === 'jwt-abc'`; `getState().admin` equals the passed identity object |
| clearAuth resets to initial state | first call `setAuth(...)` | call `useAdminAuthStore.getState().clearAuth()` | `token: null`, `admin: null` — same shape as the never-logged-in initial state |
| Persists under the exact key `admin-auth-storage` | — | call `setAuth('jwt-abc', {id, email})` | `localStorage.getItem('admin-auth-storage')` is non-null and, once parsed, its `state.token` equals `'jwt-abc'` — this exact key string is a cross-cutting contract also relied on by `axios.test.ts` below; a rename here without updating the interceptor is the single most likely regression in this file |
| `resetAdminAuthStore` test helper fully clears persisted state | `setAuth(...)`, confirm non-null `localStorage` entry | call `resetAdminAuthStore()` (per Guide 5.3) | `getState()` matches `getInitialState()`; note this helper resets in-memory state — it is `afterEach`'s job (via `resetStores.ts`), not this store's own runtime code, to guarantee this runs between every test file in the suite |

---

### 9.2 Test Case Detail — axios.test.ts

Per Guide Section 6.6, this file mocks `axios` at the library level (`vi.mock('axios')`) so the actual interceptor code under test still runs — it is the one file in the suite where mocking `@/lib/axios` itself would defeat the point. Treat as **High Scrutiny** regardless of the "utility/config" label.

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Request interceptor attaches Bearer token when `admin-auth-storage` is present | seed `localStorage` with a persisted-shape entry (`{state: {token: 'jwt-abc', admin: {...}}}`) under `admin-auth-storage` | invoke the request interceptor's handler directly with a bare config object | resulting `config.headers.Authorization === 'Bearer jwt-abc'` |
| Request interceptor sends no Authorization header when `admin-auth-storage` is absent | `localStorage` has no `admin-auth-storage` key | invoke the request interceptor's handler | resulting `config.headers.Authorization` is `undefined` — confirms a public call proceeds unauthenticated rather than the interceptor throwing or fabricating a header |
| Malformed `admin-auth-storage` JSON does not throw | seed `localStorage` with a non-JSON string under `admin-auth-storage` | invoke the request interceptor's handler | no exception propagates; `config.headers.Authorization` remains `undefined` — mirrors the try/catch "skip attaching, don't crash the request pipeline" behavior |
| Response interceptor (success) unwraps the `{statusCode, success, message, data}` envelope | mock a resolved raw response shaped `{data: {statusCode: 200, success: true, message: 'OK', data: {id: 'uuid'}}}` | pass it through the response interceptor's success handler | the handler's returned `response.data` equals `{id: 'uuid'}`, not the full envelope |
| 401 on an `/admin/*` request clears storage and redirects to the admin login route | seed `admin-auth-storage`; mock a rejected error shaped `{response: {status: 401}, config: {url: '/admin/pipeline/sources'}}`; spy/stub the redirect mechanism (`window.location.href` via a jsdom-compatible shim, per Guide's Section 6.6 approach) | pass the error through the response interceptor's error handler | `localStorage.getItem('admin-auth-storage')` is now `null`; the redirect target equals `ROUTES.ADMIN_LOGIN`, **not** a generic `/login` |
| 401 on a non-`/admin/` request does **not** clear storage or redirect | seed `admin-auth-storage`; mock a rejected error shaped `{response: {status: 401}, config: {url: '/case-studies'}}` | pass the error through the response interceptor's error handler | `localStorage.getItem('admin-auth-storage')` is unchanged (still present); no redirect occurs — this is the project's deliberate divergence from the base template's blanket 401 handler (frontend conventions 0.6); public visitors are never bounced to an admin login screen they have no reason to see |
| Non-401 errors pass through unchanged regardless of URL | mock a rejected error with `status: 500` on an `/admin/` URL | pass through the error handler | the same error is re-rejected/propagated; no storage clear, no redirect |
| Error rejection is always propagated to the caller | any of the above error cases | pass through the error handler | the handler's returned/thrown value is still a rejected promise carrying the original error — callers (hooks) still see `isError: true`, this interceptor never silently swallows a request failure |

---

### 9.2 Test Case Detail — ProtectedRoute.test.tsx

Per Guide Section 6.4 — set the store's state directly before rendering, no UI login flow.

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders the child route when a token is present | `useAdminAuthStore.setState({token: 'jwt-abc'})`; `renderRoutes([{element: <ProtectedRoute/>, children: [{path: '/', element: <div>Admin Dashboard</div>}]}])` | render | "Admin Dashboard" text is on screen |
| Redirects to admin login when no token | `useAdminAuthStore.setState({token: null})`; router includes both the protected route and a `ROUTES.ADMIN_LOGIN` route rendering known text | render at `/` | the admin-login route's content is on screen instead — asserted via resulting location/content, not via inspecting `<Navigate>` props internally |
| Store is reset between tests | (relies on the global `resetStores.ts` per 9.0) | run this file's cases in sequence | a test asserting "no token" never sees state leaked from a prior "has token" case in the same file |

---

### 9.2 Test Case Detail — PublicRoute.test.tsx

This guard wraps **only** `ROUTES.ADMIN_LOGIN` in this project (frontend conventions 0.2) — unlike a typical template where "public routes" might mean the whole app, here it means specifically "the one page an already-authenticated admin shouldn't see again."

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders the login page when no token | `useAdminAuthStore.setState({token: null})`; router has `PublicRoute` wrapping a route rendering "Admin Login" | render | "Admin Login" is on screen |
| Redirects an already-authenticated admin away from login | `useAdminAuthStore.setState({token: 'jwt-abc'})`; router includes both the `PublicRoute`-wrapped login route and a `ROUTES.ADMIN_DASHBOARD` route rendering known text | render at `ROUTES.ADMIN_LOGIN` | the dashboard route's content is on screen instead, confirming redirect target is `ROUTES.ADMIN_DASHBOARD`, not the generic template's `ROUTES.HOME` — a copy-paste of the template's default `PublicRoute` (which redirects to `HOME`) would fail this case |

---

### 9.2 Test Case Detail — PublicLayout.test.tsx

Per Guide 6.1's layout rule (assert the layout renders its `<Outlet/>` children), plus a project-specific negative case: `PublicLayout` is explicitly **not** auth-aware (frontend conventions 0.1).

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders `<Outlet/>` child content | `createMemoryRouter([{element: <PublicLayout/>, children: [{path: '/', element: <div>Map Page</div>}]}])` | render | "Map Page" text is on screen, inside the layout's nav/footer chrome |
| Renders identically regardless of admin auth state | render once with `useAdminAuthStore.setState({token: null})`, once with `{token: 'jwt-abc'})` | compare rendered nav/footer content between the two | no difference in rendered output — confirms the component genuinely reads nothing from `adminAuth.store`; a regression where someone "helpfully" added an admin-aware nav link here wouldn't be caught by the first case alone |

---

### 9.2 Test Case Detail — DashboardLayout.test.tsx

| Case | Setup | Action | Expected result |
|---|---|---|---|
| Renders `<Outlet/>` child content | `createMemoryRouter([{element: <DashboardLayout/>, children: [{path: '/', element: <div>Pipeline Overview</div>}]}])` | render | "Pipeline Overview" text is on screen |
| Renders nav links to all four admin routes | render | inspect nav | links present targeting `ROUTES.ADMIN_DASHBOARD`, `ROUTES.ADMIN_PIPELINE`, `ROUTES.ADMIN_WEIGHT_CONFIGS`, `ROUTES.ADMIN_CASE_STUDIES` — all four, not a subset |
| Logout action calls the logout mutation, not a direct store clear | `vi.mock('@/hooks/useAdminAuth')`; mock `useAdminLogout` to return `{mutate: vi.fn(), isPending: false}` | `userEvent.click` the "Logout" control | the mocked `mutate` was called — asserting via `vi.mocked(...).mock.calls`, not merely that a button with the word "Logout" exists in the DOM |
| Nav renders with no live/loading data | render with no query mocks configured at all for this component | render | no loading indicator or badge count appears in the nav — confirms the nav is static markup, unlike a cart-badge-style layout, per Doc 7.7's note |

---

### 9.2 Test Case Detail — index.test.tsx (route composition)

Feeds the real, exported `routeObjects` (see 9.0's testability note) into a scoped `createMemoryRouter`, using the real `ProtectedRoute`/`PublicRoute`/`PublicLayout`/`DashboardLayout`/`AuthLayout` components — this is the one place in the suite that intentionally composes several already-unit-tested pieces together, to catch a wiring mistake unit tests in isolation can't.

| Case | Setup | Action | Expected result |
|---|---|---|---|
| `/` renders LandingPage inside PublicLayout, no guard | `useAdminAuthStore.setState({token: null})`; `renderRoutes(routeObjects, ['/'])` | render | landing page content visible; no redirect of any kind occurs for an anonymous visitor |
| `/map`, `/case-studies`, `/case-studies/:id`, `/methodology` all render without a token | `token: null` | render at each path in turn | each page's content is visible — none of the public routes ever redirect regardless of auth state |
| `/admin/login` renders the login page when unauthenticated | `token: null` | `renderRoutes(routeObjects, ['/admin/login'])` | admin login page content visible |
| `/admin/login` redirects to the dashboard when already authenticated | `token: 'jwt-abc'` | render at `/admin/login` | dashboard content visible instead |
| Every `/admin/*` route (except `/admin/login`) redirects to login when unauthenticated | `token: null` | render at `/admin`, then separately at `/admin/pipeline`, `/admin/weight-configs`, `/admin/case-studies` | each one resolves to the login page's content, not the intended admin page — asserted per-route, not inferred from testing only one of the four |
| Every `/admin/*` route renders its intended page when authenticated | `token: 'jwt-abc'` | render at each of the four admin paths | each resolves to its own distinct page content (Dashboard/Pipeline/WeightConfig/CaseStudyCuration), not all four resolving to the same fallback content by accident |
| Unknown path renders NotFoundPage | — | render at `/this-does-not-exist` | NotFoundPage content visible, with no layout chrome around it (per the template's own "no layout" default for the catch-all) |

---

### 9.4 Coverage Honesty Check (per PR Steward, at review time)

Use this checklist, don't just trust the green checkmark.

- [ ] The 401-scoping test in `axios.test.ts` genuinely exercises **both** an `/admin/` URL and a non-`/admin/` URL as two separate cases with different expected outcomes — a test that only covers the `/admin/` case and asserts "not equal to some other case" by inference would miss a regression where the scoping check itself is missing entirely
- [ ] Every `admin-auth-storage` key-name assertion in `axios.test.ts` and `adminAuth.store.test.ts` uses the literal string, not a local constant re-declared inside the test file that could silently drift from the real `persist({name: ...})` config
- [ ] `ProtectedRoute`/`PublicRoute` tests actually import and exercise the real `useAdminAuthStore` (renamed from the template's `auth.store.ts`) — not a stale mock referencing a `useAuthStore`/`auth.store` path that no longer exists in this project, which would pass for the wrong reason
- [ ] `index.test.tsx` builds its `createMemoryRouter` from the real, imported `routeObjects` export — not a hand-typed copy of the route tree pasted into the test, which could drift from `src/routes/index.tsx` without the test ever failing
- [ ] `PublicLayout`'s "no auth-awareness" case actually varies the store's state between two render calls and diffs the output — not a single render with an unasserted-upon default state
- [ ] `DashboardLayout`'s logout case asserts the mocked `mutate` function was called (and, ideally, with what — even if with no args), not merely that a "Logout"-labeled element exists somewhere in the DOM
- [ ] The redirect-target assertions for `PublicRoute` (→ `ROUTES.ADMIN_DASHBOARD`) and the 401 handler (→ `ROUTES.ADMIN_LOGIN`) check the *specific* route constant, not just "a redirect happened somewhere" — this project deliberately diverges from the base template's generic `/login`/`HOME` targets in both places, and a loose assertion would pass even if that divergence regressed back to the template default

---

### 9.5 Out of Scope for Automated Testing (and why)

- **`src/types/index.ts`** — type-only file, no runtime behavior to exercise. Per the Testing Guide's Scrutiny Tiers (Section 12), type-only files rely on `tsc` and lint, not a dedicated test file. Its correctness is exercised indirectly everywhere a type is used — e.g. a hook test asserting the exact shape of a mocked response, a guard test asserting the exact `AdminIdentity` fields.
- **`src/constants/index.ts`** — plain `ROUTES`/`QUERY_KEYS` value objects, no logic. Same Low-scrutiny carve-out as types; its values are exercised indirectly by every test elsewhere in this suite that asserts against a specific route or query key (e.g. `PublicRoute.test.tsx`'s redirect-target assertion, or any feature hook's exact `queryKey` array assertion).
- **`src/components/layouts/AuthLayout.tsx`** — unmodified, used as-is from the base template (frontend conventions 0.1 / Doc 6). Per the same precedent as the backend's `app.ts` ("requires zero deviation... no new test surface is introduced here"), this file introduces no project-specific behavior to test; the template's own coverage (if any) already applies.
- **Real browser navigation and localStorage persistence across an actual page reload** — `createMemoryRouter` and jsdom's `localStorage` are exercised directly throughout this module, but real `BrowserRouter`/URL-bar behavior and true page-reload rehydration timing need manual QA or a future E2E pass, consistent with the Guide's own out-of-scope stance on real browser navigation (Section 15 practice exercise notes, and the analogous eco-9.2.1 auth-pages precedent).
- **Loaders/actions or any other `createBrowserRouter` data-router feature beyond route composition** — this project's critical routing update adopts `createBrowserRouter` + `RouterProvider` for its API shape only; per frontend conventions and the template's own philosophy (simple client-rendered SPA, no data-loading-on-navigation pattern), no route in this app uses a `loader`/`action`, so there is nothing of that kind to test here or in any feature module.
- **Visual/responsive rendering of `PublicLayout`/`DashboardLayout`** — presence, structure, and text content are asserted; pixel-level layout and true mobile-viewport rendering (FR-26/FR-27's actual visual requirement) is a manual/visual-QA concern, not a unit-test one.

---

**Next:** proceed to → [9-2. Frontend Test Documentation: Growth Map & Exploration]
