## Project: Shadow Economy Map — Frontend UI Foundation Spec
**Source:** Stitch-generated design (`shadow_economy_map/DESIGN.md` + 10 pages of `code.html`/`screen.png`)
**Depends on:** `frontend-core-arch-doc.md` (template-react v3.1), `0-frontend-conventions.md`
**Read this before:** any page or feature component. This is the foundation every page in `01`–`06` and `8-2`–`8-7` builds on top of.

---

### 9.0 Why This Doc Exists

The Stitch design is pixel-consistent across all 10 pages — same color tokens, same typography scale, same nav/sidebar/footer patterns repeated verbatim. This doc extracts that shared system once, so it's built once (per template Section 1.3's own DRY discipline) instead of copy-pasted per page. Everything below was verified directly against the Stitch `code.html` files and `DESIGN.md`, not assumed.

**Deviation from the base template, called out explicitly:** the arch doc (Section 5.2) specifies `lucide-react` as the icon library. The Stitch design uses **Material Symbols Outlined** (Google Fonts) throughout, with no `lucide-react` icons anywhere in any page. Per the user's explicit decision, this project uses Material Symbols for pixel fidelity — `lucide-react` should be removed from `package.json` rather than left as an unused dependency (same dependency-hygiene rule the arch doc applies to `tailwindcss-animate`, Section 2.2).

---

### 9.1 Design Tokens → `src/styles/globals.css`

Replace the template's generic neutral OKLCH theme entirely — these are real project values, not placeholders.

**Colors** (hex, as given by `DESIGN.md` — kept as hex rather than converted to OKLCH, since these are the actual verified brand values, not shadcn's generated defaults):

| Token | Value | Token | Value |
|---|---|---|---|
| `background` / `surface` | `#f7f9fe` | `on-surface` | `#181c20` |
| `surface-dim` | `#d8dadf` | `on-surface-variant` | `#424754` |
| `surface-container-lowest` | `#ffffff` | `outline` | `#727785` |
| `surface-container-low` | `#f1f4f9` | `outline-variant` | `#c2c6d6` |
| `surface-container` | `#eceef3` | `primary` | `#0057c0` (buttons render `#3F82F7`, hover `#3275EC` — see 9.6) |
| `surface-container-high` | `#e6e8ed` | `on-primary` | `#ffffff` |
| `surface-container-highest` | `#e0e2e7` | `primary-container` | `#2670e4` |
| `secondary` | `#585f67` | `on-primary-container` | `#fefcff` |
| `secondary-container` | `#dce3ed` | `secondary-fixed` | `#dce3ed` (admin sidebar bg) |
| `tertiary` | `#4c5c7d` | `secondary-fixed-dim` | `#c0c7d1` |
| `error` | `#ba1a1a` | `on-secondary-fixed-variant` | `#40474f` |
| `error-container` | `#ffdad6` | `on-error-container` | `#93000a` |
| `text-secondary` | `#52627A` | `text-muted` | `#8290A7` |
| `border-base` | `#E6EBF4` | | |

**Heatmap palette** (map-specific, `components/map/GrowthMapCanvas.tsx` only): `heatmap-low #69C79A` · `heatmap-med-low #9BD6A9` · `heatmap-med #F3D96A` · `heatmap-high #F5A34A` · `heatmap-extreme #E74F3D`

**Status colors** (used for badges/chips — see 9.4 `StatusBadge`; found hardcoded inline across the design instead of tokenized, normalize here): success `#43B982` on `#EAF8F2`, warning `#F5A34A` on `#FFF3E0`/`#FFF4E5`, error/failed `#E74F3D` on `#FCE8E8`.

**Typography** — all `Inter`, loaded via Google Fonts in `index.html`:

| Name | Size | Weight | Line-height | Letter-spacing |
|---|---|---|---|---|
| `hero-lg` | 48px | 800 | 1.1 | -0.03em |
| `hero-lg-mobile` | 32px | 800 | 1.2 | -0.02em |
| `page-title` | 32px | 700 | 1.15 | -0.01em |
| `card-title` | 16px | 700 | 1.4 | — |
| `body-md` | 15px | 400 | 1.6 | — |
| `body-sm` | 13px | 400 | 1.5 | — |
| `label-caps` | 11px | 600 | 1 | 0.06em (rendered uppercase) |
| `data-kpi` | 17px | 700 | 1 | — |

**Spacing** (8px rhythm): `micro 4px · sm 8px · md 16px · lg 24px · xl 48px · xxl 64px · gutter 24px · margin-mobile 16px · margin-desktop 48px · max-width 1320px`

**Radii**: `sm 0.25rem · DEFAULT 0.5rem (buttons/inputs) · md 0.75rem · lg 1rem (16px — standard cards) · xl 1.5rem · full 9999px (pills/badges)`. Exception: heatmap cells use a near-sharp `2px` radius, not the standard card radius — set locally in `GrowthMapCanvas`, not a global token.

**Shadows** (define once as utility classes in `globals.css`, not per-page `<style>` blocks as Stitch has them):
```css
.shadow-card, .soft-shadow { box-shadow: 0 12px 35px rgba(50, 88, 150, 0.08); }
.shadow-nav { box-shadow: 0 2px 10px rgba(50, 88, 150, 0.05); }
.glass-panel {
  background: rgba(255, 255, 255, 0.82);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
}
```
`glass-panel` is used for map popovers/AI-summary overlays (`CellDetailPanel`) and any contextual floating card, per `DESIGN.md`'s Elevation section.

Map these into Tailwind v4's `@theme` block in `globals.css` (CSS-first, no `tailwind.config.ts`, per arch doc Section 4.3) exactly as the template's existing token structure does for shadcn's palette — same mechanism, different values.

---

### 9.2 `index.html` Changes

Add both font links (Stitch uses both on every page):
```html
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700;800&display=swap" rel="stylesheet">
<link href="https://fonts.googleapis.com/css2?family=Material+Symbols+Outlined:wght,FILL@100..700,0..1&display=swap" rel="stylesheet">
```
Icons are used as `<span class="material-symbols-outlined">icon_name</span>`, stroke weight 2px equivalent (default), filled variant (`FILL 1`) only for a small number of emphasis cases (e.g. logo mark, active nav icon) — match each page's existing usage rather than filling everything by default.

---

### 9.3 `components/ui/` — shadcn Primitives Needed

Only what the 10 pages actually use (not the full shadcn catalog): `Button`, `Card` (+ `CardContent`), `Input`, `Label`, `Checkbox`, `Badge`. All themed automatically once `globals.css` tokens (9.1) are in place — no per-component color overrides needed since shadcn/ui components consume the CSS variables directly.

---

### 9.4 `components/common/` — Shared Components

| Component | Responsibility | Used by |
|---|---|---|
| `TopNavBar.tsx` | Public nav: logo, Map/Case Studies/Methodology/About links with active-link styling (`useLocation()`-derived — the active link gets `text-primary border-b-2 border-primary`, others `text-on-secondary-container`), "Explore the Map" CTA, mobile hamburger toggle | `PublicLayout` |
| `Footer.tsx` | Logo, copyright line, Terms/Privacy/Data Sources/API Documentation links (static, no routes exist yet for these — render as `#` placeholders) | `PublicLayout` |
| `AdminSidebar.tsx` | Fixed 220px sidebar: admin avatar + "System Health: Optimal" line, nav (Pipeline/Weight Configs/Case Studies) with active-link styling, Logout action calling `useAdminLogout().mutate()`. **No "Settings" link** — present in the Stitch design but not backed by any route or spec in `04`–`06`; deliberately left out until a real settings page/route exists | `DashboardLayout` |
| `StatusBadge.tsx` | Pill badge, `{ status: 'success' \| 'warning' \| 'error' \| 'neutral'; label: string }` — consolidates the success/warning/error chip pattern repeated with hardcoded colors across `SourceHealthCard`, `PipelineRunsTable`, evidence-tier badges on case study cards | `admin-pipeline/*`, `case-studies/*` (evidence tier), `EmptyState` n/a |
| `EmptyState.tsx` | Already spec'd in `06`/`8-3` — generic empty message + optional CTA | `CaseStudiesListPage`, `CaseStudyTable`, `WeightConfigHistory` |

Both `TopNavBar` and `AdminSidebar` contain real conditional logic (active-route detection), which is the reason they're separated from their layouts rather than inlined — matches the template's own "extract when a page/component is getting large or has real logic" rule (Section 11.1), applied one level up to layouts.

The mobile header bar inside `DashboardLayout` (hamburger toggle, shown only on small screens) stays inline in the layout — trivial, layout-specific, no logic worth isolating.

---

### 9.5 Layouts

| Layout | Composition | Guard |
|---|---|---|
| `PublicLayout.tsx` | `TopNavBar` + `<Outlet />` + `Footer` | None (per conventions 0.1) |
| `AuthLayout.tsx` | Used as-is from template — split panel: decorative image (desktop only) left, form slot right. No project-specific change needed; Stitch's admin login page fits this shape directly | `PublicRoute` |
| `DashboardLayout.tsx` | `AdminSidebar` (desktop) + inline mobile header (mobile) + `<Outlet />`, both wrapping a `max-w-max-width mx-auto` content area | `ProtectedRoute` |

---

### 9.6 Component Style Notes (for `ui/` theming + common components)

- **Button primary**: solid `#3F82F7`, hover `#3275EC` + 1px lift + shadow increase, white text, `rounded-lg` (8px), uppercase `label-caps` text
- **Button secondary**: `#EAF2FF` background, primary-blue text, no border
- **Input**: white bg, 1px `border-base` border, `rounded-lg`, focus → primary-blue border + 2px soft blue ring
- **Card**: white bg, `rounded-xl` (16px), `p-lg` (24px) internal padding, `.soft-shadow`
- **Badge/chip**: `rounded-full`, soft background + base-color text (see status colors, 9.1)

---

### 9.7 Foundation Build Order

Strict dependency order — nothing below a line can be built before the line above it:

1. `src/styles/globals.css` (tokens — everything depends on this)
2. `index.html` (font links)
3. `src/lib/utils.ts` — `cn()`, unchanged from template
4. `components/ui/`: `Button`, `Card`, `Input`, `Label`, `Checkbox`, `Badge`
5. `components/common/StatusBadge.tsx`, `EmptyState.tsx` (depend on `ui/` + tokens only)
6. `components/common/TopNavBar.tsx`, `Footer.tsx`, `AdminSidebar.tsx` (depend on `ui/`, tokens, and routing constants — `ROUTES` from `src/constants/index.ts` must exist first, see `8-1`)
7. `components/layouts/PublicLayout.tsx`, `DashboardLayout.tsx`, `AuthLayout.tsx` (depend on step 6)

This entire sequence must complete before any page in `7b`'s §7.7 build order (step 5 onward there) can render correctly end-to-end.

---

**Next:** proceed to → page/feature implementation per `7b-frontend-file-structure.md` §7.7
