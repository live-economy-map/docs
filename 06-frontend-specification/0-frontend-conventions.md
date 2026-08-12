## Project: Shadow Economy Map — Frontend Specification
**Links back to:** [1. Problem & Solution Statement], [2. Functional & Non-Functional Requirements], [3. Use Cases], [4. Database & Data Model], [API Specification], [roadmap.md]
**Built on:** template-react v3 (React 19, Vite, TypeScript, Tailwind v4, shadcn/ui, TanStack Query, Axios, Zustand, React Router v7, React Hook Form + Zod)

Split into feature-based files, mirroring the same grouping used for requirements, use cases, and the API spec:

- **0-frontend-conventions.md** — this file
- **01-growth-map-frontend.md** — Growth Map & Exploration
- **02-case-studies-frontend.md** — Case Studies & Validation
- **03-site-content-frontend.md** — Site Content & Onboarding
- **04-admin-access-frontend.md** — Admin Access
- **05-data-pipeline-frontend.md** — Data Pipeline Management
- **06-case-study-curation-frontend.md** — Case Study Curation

---

### 0.1 The Core Adaptation: Public-First, Not Auth-First

The template assumes an app that's entirely behind login, with only a login page public. **Our app is the inverse:** the large majority is public with zero auth, and only `/admin/*` is protected. This changes how routing, layouts, and guards are used — everything else in the template (Axios contract, TanStack Query, Zustand, forms) applies unchanged.

**Three route categories, not the template's two:**

| Category | Guard | Layout | Applies to |
|---|---|---|---|
| Public (no auth concept at all) | None | `PublicLayout` (new — not in base template) | Map, Case Studies, Site Content — the majority of the app |
| Admin login | `PublicRoute` (redirect away if already logged in) | `AuthLayout` (from template, used as-is) | `/admin/login` only |
| Admin area | `ProtectedRoute` (from template, used as-is) | `DashboardLayout` (from template, used as-is) | Everything under `/admin/*` except `/admin/login` |

`PublicLayout` is a new addition to the template's `components/layouts/` — a thin wrapper with a public nav/footer, no auth awareness at all. It does not exist in the base template because the base template assumes nothing is public.

---

### 0.2 Routing Structure

```
/                          → PublicLayout → Landing Page
/map                       → PublicLayout → Growth Map Page
/case-studies              → PublicLayout → Case Studies List Page
/methodology               → PublicLayout → Methodology Page
/admin/login               → PublicRoute → AuthLayout → Admin Login Page
/admin                     → ProtectedRoute → DashboardLayout → Admin Dashboard (pipeline overview)
/admin/pipeline             → ProtectedRoute → DashboardLayout → Pipeline Management Page
/admin/weight-configs        → ProtectedRoute → DashboardLayout → Weight Config Page
/admin/case-studies          → ProtectedRoute → DashboardLayout → Case Study Curation Page
*                          → NotFoundPage (no layout, per template default)
```

Per the template's own rule (Section 20.2): every new route must be nested under the correct guard **and** the correct layout — never bare. For public routes, "correct guard" means explicitly *no* guard component, not an omission.

---

### 0.3 State Management Split

- **Zustand:** only `adminAuth.store.ts` (renamed from the template's generic `auth.store.ts` to be explicit that only admins authenticate in V1) — persists under key `admin-auth-storage`. No other Zustand store exists in V1; there is no client-side state for public visitors that needs to persist or be global (map view state like current period/layer is local `useState`/URL params, not global store — it doesn't need to survive a page reload the way auth does).
- **TanStack Query:** every server call, public or admin, per the template's standard pattern — one hook file per feature, matching the API spec's 6-feature split exactly.

---

### 0.4 API Layer

Uses the template's `src/lib/axios.ts` unchanged — our backend's `SuccessResponse`/`ErrorResponse` envelope (per API Conventions 0.1) matches the template's expected contract exactly, so the existing interceptor unwrap requires no modification.

`VITE_API_URL` must be set to our backend's base path, e.g. `http://localhost:3000/api/v1`, per the template's env contract (Section 6.1).

**One addition beyond the base template:** the Axios instance attaches the admin Bearer token (from `adminAuth.store.ts`) via a request interceptor **only when present** — public endpoints are called with the same `api` instance, simply without a token, rather than needing a second unauthenticated client. The backend doesn't require a header on public routes, so an absent token on a public call is a non-issue.

---

### 0.5 Types & Query Keys

`src/types/index.ts` gets one interface block per feature, matching the API spec's response shapes exactly (e.g., `GrowthCell`, `CellDetail`, `CaseStudy`, `PipelineRun`, `ScoreWeightConfig`) — field names and types copied directly from the API spec's example JSON, not re-derived.

`src/constants/index.ts` `QUERY_KEYS` gets one entry per feature's queryable resources: `MAP_CELLS`, `CELL_DETAIL`, `MAP_LAYER`, `CASE_STUDIES`, `CASE_STUDY_DETAIL`, `CONTENT_LANDING`, `CONTENT_METHODOLOGY`, `PIPELINE_SOURCES`, `PIPELINE_RUNS`, `WEIGHT_CONFIGS`, `ADMIN_CASE_STUDIES`.

---

### 0.6 401 Handling — One Deliberate Divergence from the Template

The template's interceptor hard-redirects to `/login` on any `401`. **For us, a `401` can only ever come from an `/admin/*` call** (public endpoints never return `401`, per the API Conventions doc). The interceptor should only trigger the redirect-to-login behavior when the failed request's URL matches `/admin/`, and should redirect to `/admin/login` specifically — a `401` should never redirect a public visitor browsing the map to any kind of login page, since public visitors never log in at all.

---

### 0.7 Component Split (per template Section 20.2, applied to our features)

- `components/ui/` — shadcn/ui primitives, untouched, no app logic (template default)
- `components/common/` — shared, feature-agnostic pieces (e.g., `ErrorBoundary`, a generic `EmptyState`)
- `components/layouts/` — `PublicLayout` (new), `DashboardLayout`, `AuthLayout` (both from template)
- `components/map/`, `components/case-studies/`, `components/admin-pipeline/`, etc. — one folder per feature for feature-specific components (e.g., `components/map/GrowthMapCanvas.tsx`, `components/map/CellDetailPanel.tsx`, `components/map/TimeSlider.tsx`), following the same "app-specific goes in its own folder, not into `ui/`" rule the template already enforces.

---

**Next:** proceed to → [01. Growth Map & Exploration Frontend]
