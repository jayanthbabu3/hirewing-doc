# Architecture Review — SKU Viewer (Digital Catalogue)

**Date:** 2026-06-12
**Stack reviewed:** React 18 + Vite, Material UI v6, React Router v6 (~3,800 lines of source)

This document answers three questions:

1. [Material UI vs Tailwind CSS — which should we use?](#1-material-ui-vs-tailwind-css)
2. [Is the project structure good? What should change?](#2-project-structure-review)
3. [Redux vs Context API — which should we use?](#3-redux-vs-context-api)

---

## 1. Material UI vs Tailwind CSS

### Recommendation: **Stay with Material UI**

The migration to MUI is already complete:

- `package.json` contains `@mui/material`, `@mui/icons-material`, `@mui/x-data-grid`
- Every component uses MUI imports
- No Tailwind config, classes, or dependencies remain in the source

Going back to Tailwind would be pure rework with no functional gain.

### How this fits current trends

| | Tailwind CSS (+ shadcn/ui) | Material UI |
|---|---|---|
| **Best for** | Marketing sites, design-heavy consumer products, custom design systems | Internal tools, admin dashboards, data-heavy enterprise apps |
| **Components** | You build them (or use shadcn/ui) | Rich library out of the box (DataGrid, Drawer, Menu, Autocomplete…) |
| **Team skill required** | Strong CSS fundamentals | Component API knowledge |
| **Theming** | Design tokens via config | Centralized theme object |

Tailwind dominates new greenfield consumer projects, but **MUI is still the standard choice for exactly this kind of app** — an internal, data-heavy catalogue/admin tool with tables, filters, drawers, and role-based menus. Combined with the team being more comfortable with MUI, the decision is clear.

### Cleanup to do (MUI-related)

Components are full of hardcoded hex values inside `sx` props — `#9ca3af`, `#d1d5db`, `#b71c1c` (these are leftover Tailwind gray-palette colors). Repeated styles like `fontSize: '0.813rem', fontWeight: 700` appear everywhere.

**Action:** Move colors and typography into `src/theme.js` (palette + component overrides via `createTheme`) so styling stays consistent and components stay lean.

---

## 2. Project Structure Review

### Current structure

```
src/
├── App.jsx            (496 lines — layout + nav + state + routing)
├── main.jsx
├── context.jsx
├── theme.js
├── components/        (pages and shared components mixed together)
│   ├── ListPage.jsx   (718 lines)
│   ├── DetailPage.jsx
│   ├── AdminPage.jsx
│   ├── InventoryTab.jsx
│   ├── BigCommerceTab.jsx
│   └── ui.jsx
├── constants/
└── utils/
```

The `components/` / `constants/` / `utils/` split is a sensible foundation for the current size. The issues below are ordered by priority.

### 🔴 Priority 1 — Security: API tokens in client code

`src/constants/api.js` builds `BC_HEADERS` (BigCommerce auth headers) directly in frontend code. **Any token shipped in client-side JavaScript is visible to every user** via DevTools or the bundled `dist/` output.

There are also **two `.env` files** — one at the project root and one inside `src/`. Vite only reads env files from the project root; files inside `src/` risk being bundled and served.

**Actions:**
- Route BigCommerce calls through a small backend proxy (serverless function or tiny Express server) that holds the tokens.
- Delete `src/.env`; keep a single root `.env` with `VITE_`-prefixed variables only for genuinely public values.

### 🟠 Priority 2 — Break up the two giant files

**`App.jsx` (496 lines)** is the layout, sidebar, top bar, user menu, routing, *and* the global state owner. Extract:

```
src/
├── layout/
│   ├── Sidebar.jsx      (NavItem, GroupLabel, drawer logic)
│   ├── TopBar.jsx       (AppBar, breadcrumb)
│   └── UserMenu.jsx     (avatar menu, switch-user)
├── pages/
│   ├── ListPage.jsx
│   ├── DetailPage.jsx
│   └── AdminPage.jsx
├── components/          (shared building blocks only)
│   ├── ui.jsx
│   ├── InventoryTab.jsx
│   └── BigCommerceTab.jsx
```

`App.jsx` then becomes just providers + routes + layout shell.

**`ListPage.jsx` (718 lines)** has ~18 `useState` hooks. Split the API-search panel, results table, and pagination into child components, and collapse the related `api*` state group (`apiAttr`, `apiOp`, `apiKw`, `apiMax`, `apiFetching`, `apiError`, …) into a single custom hook such as `useEntitySearch()`.

### 🟡 Priority 3 — Minor cleanup

- Remove the leftover debug effect in `App.jsx` (~line 297): `useEffect(() => console.log("listpayload", listPayload), [listPayload])` — `listPayload` is an imported constant, so the effect is pointless.
- The project folder name `catalouge` is misspelled (`catalogue`) — cosmetic, but fix it before it spreads into repo names or URLs.

---

## 3. Redux vs Context API

### Recommendation: **Context API — Redux is not needed**

The global state in this app is small and changes infrequently:

- Selected/context entity
- Active tab / inventory sub-tab
- Locale
- Current user, role, permissions
- App config

Redux (even Redux Toolkit) would add boilerplate and a learning curve for state a single context already handles. This matches the broader trend: new apps reach for Context (or a tiny store like Zustand) for client state, and reserve Redux for apps with lots of complex, frequently-updated, cross-cutting state — which this isn't.

### ⚠️ But fix the current Context implementation

In `App.jsx` (~line 200), `contextValue` is **rebuilt as a new object on every render and never memoized**. Every keystroke or sidebar toggle re-renders *every* consumer of `AppContext`.

**Fix (do this now):**

```jsx
const contextValue = useMemo(() => ({
  selectedEntity, setSelectedEntity,
  contextEntity, setContextEntity,
  activeTab, setActiveTab,
  // ...
}), [selectedEntity, contextEntity, activeTab, invSubTab,
     locale, config, currentUser, permissions, isAdmin]);
```

Wrap the handler functions (`handleSelect`, `onSearch`, `onHighlight`) in `useCallback` so they don't invalidate the memo.

**If the app grows:** split into two contexts so changing a tab doesn't re-render permission-driven UI:

- `SessionContext` — stable: user, permissions, config, locale
- `SelectionContext` — volatile: selected entity, active tab, sub-tab

### Server data is the real future pain point — use TanStack Query, not Redux

The manual `fetch` + `thumbnailCache` code in `utils/api.js`, and the loading/error `useState` clutter in `ListPage`/`DetailPage`, are server-state concerns. If fetching, caching, or retries become painful, the right tool is **TanStack Query (React Query)** — it replaces most of that hand-rolled state and caching. Redux would not solve this problem.

---

## Summary

| Question | Decision | Key action |
|---|---|---|
| MUI vs Tailwind | **Keep MUI** — already migrated, right fit for an internal data tool | Move hardcoded colors/typography into `theme.js` |
| Project structure | **Good foundation, needs changes** | Move API tokens to a backend proxy; split `App.jsx` and `ListPage.jsx`; add `pages/` + `layout/` folders |
| Redux vs Context | **Context API** — app doesn't justify Redux | Memoize `contextValue` with `useMemo`/`useCallback`; adopt TanStack Query later for server data |
