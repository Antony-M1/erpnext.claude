---
name: frappe-vue-3
description: >
  Use this skill whenever working on a Vue 3 single-page app inside a Frappe
  app's `frontend/` directory (the frappe-ui + Vite pattern used by Frappe
  CRM, Helpdesk, and this app's own `apps/fms/frontend` Marketplace Portal).
  Trigger it for: writing or editing `.vue` components under `frontend/src`;
  touching `vite.config.js`, `frontend/package.json`, `main.js`, `App.vue`,
  `router.js`, Pinia stores, or composables; anything about the `yarn dev`
  vs `yarn build` workflow, the Vite dev server port, or the `www/<app>/*`
  controller + `get_context_for_dev` boot-data pattern; and Chrome/Vue
  "Devtools inspection is not available — production mode" warnings on a
  Frappe-hosted Vue app. Also trigger on requests like "add a component to
  the portal", "why isn't Vue devtools working", "wire up a new page/route
  in the frontend", or "add a Pinia store".
tools:
  - Read
  - Write
  - Edit
  - Bash
---

# Frappe + Vue 3 (frappe-ui / Vite) Skill

This skill packages the architecture and conventions for the Vue 3 SPA
pattern used in Frappe apps built on `frappe-ui` + Vite (the same pattern
Frappe CRM and Helpdesk use). It's grounded in this repo's own instance:
`apps/fms/frontend` (the FMS Marketplace Portal). **Only use the information
in this file to answer questions about this pattern — if something isn't
covered here, say so and inspect the specific app's `frontend/vite.config.js`
/ `package.json` / `main.js` rather than guessing.**

For custom & installed app details, refer to the project's `CLAUDE.md` first.

## Source documentation (canonical)

- frappe-ui components/docs: https://frappeui.com
- Vite: https://vitejs.dev/config/
- Vue 3: https://vuejs.org/guide/introduction.html
- Vue 3 devtools flags: https://vuejs.org/api/compile-time-flags.html (`__VUE_PROD_DEVTOOLS__`)

---

## 1. Directory layout (frontend/)

A frappe-ui Vue 3 SPA under `apps/<app>/frontend/` follows this shape:

| Path | Purpose |
|---|---|
| `vite.config.js` | Vite config; always wraps the `frappe-ui/vite` plugin (`frappeProxy`, `lucideIcons`, `jinjaBootData`, `buildConfig`) |
| `package.json` | `dev` (`vite`), `build` (`vite build --base=... && copy-html-entry`), `serve` (`vite preview`) scripts |
| `index.html` | Vite entry HTML — only used as-is in `yarn dev`; `yarn build` copies its built version into the Frappe app's `www/<route>/*.html` |
| `src/main.js` | Creates the Vue app, installs Pinia/router/FrappeUI/translation plugin, mounts to `#app` |
| `src/App.vue` | Root component — layout switching, global providers (`FrappeUIProvider`, `Dialogs`) |
| `src/router.js` | `vue-router` routes, route guards (auth/role checks) |
| `src/config/*.js` | Single source of truth for constants shared with the backend (route base paths, enum-like maps) — never hardcode a route string inline |
| `src/pages/*.vue` | Route-level components (one per `router.js` entry) |
| `src/components/**/*.vue` | Reusable presentational/composite components |
| `src/composables/*.js` | Shared reactive logic (`useX()` functions) |
| `src/stores/*.js` | Pinia stores for cross-page state (e.g. session, theme) |

Backend side (in the Frappe app, not `frontend/`):

| Path | Purpose |
|---|---|
| `<app>/www/<route>/<name>.py` | `no_cache = 1` controller; `get_context(context)` sets `context.boot` — this is what Jinja injects as `window.*` globals for the **built** app |
| same file, `get_context_for_dev()` | A `@frappe.whitelist(allow_guest=True)` twin of the same boot payload, fetched over AJAX **only** by `yarn dev` (raw `index.html` has no Jinja templating there). Must be guarded by `frappe.conf.developer_mode` — never let this run against a production site. |

---

## 2. The two run modes — know which one you're looking at

| | `yarn dev` | `yarn build` |
|---|---|---|
| Command | `vite` (dev server) | `vite build --base=... && copy-html-entry` |
| Vite `mode` | `development` (default for `vite` serve) | `production` (default for `vite build`, no `--mode` flag passed) |
| Served from | Vite's own port (see formula below), proxying API calls to bench | `apps/<app>/public/frontend/*`, copied into `<app>/www/<route>/*.html`, served by bench on the normal site port |
| Boot data | Fetched via `get_context_for_dev()` AJAX call, `developer_mode`-gated | Injected server-side by Jinja via `context.boot` |
| HMR | Yes | No |
| Vue devtools | **Always available** (dev build) | **Compiled out** by default (`__VUE_PROD_DEVTOOLS__` defaults to `false`) |

**Vite dev server port formula** (from `frappe-ui/vite/frappeProxy.js`):
`vite_port = 8080 + (webserver_port - 8000)`. For the default bench
(`webserver_port: 8000`), that's port `8080`. Check
`sites/common_site_config.json` for the actual `webserver_port` before
assuming 8080.

To develop with hot reload + devtools, always run `yarn dev` from the app's
`frontend/` directory and browse the **Vite port**, e.g.
`http://<site>:8080/<portal-route>/...` — not the bench port. Opening the
bench port (8000) always shows the last `yarn build` output.

---

## 3. Chrome "Devtools inspection is not available — production mode"

This exact warning means you're looking at a production Vue build (see table
above) — it is **not** a bug, and it is **not** controlled by
`developer_mode` on the site. Vue 3 has no runtime `app.config.devtools`
toggle; support is purely the compile-time constant `__VUE_PROD_DEVTOOLS__`
(default `false`), baked in at bundle time.

Fix, in order:

1. First check: are you on the bench port, or the Vite dev port? If bench
   port — run `yarn dev` and switch to the Vite port. This alone fixes it
   with zero code changes.
2. If you specifically need devtools inside a **built** bundle (e.g. testing
   something that only reproduces outside the Vite dev server), make the
   flag explicit and dev-gated in `vite.config.js`, keyed off Vite's own
   `mode`:
   ```js
   export default defineConfig(async ({ mode }) => {
     const isDev = mode === 'development'
     const config = {
       // ...
       define: {
         __VUE_PROD_DEVTOOLS__: JSON.stringify(isDev),
       },
     }
     // ...
   })
   ```
   `mode` is `'production'` for a plain `yarn build`/`vite build` (no
   `--mode` flag), so this changes nothing for real deployments — it only
   ever flips `true` for an explicit `vite build --mode development` (or
   `vite --mode development`) invocation.

**Never** hardcode `__VUE_PROD_DEVTOOLS__: true` (or otherwise make it
unconditional) in a config that's also used for the real production build —
that ships devtools introspection (component tree, store contents, Pinia
state) to end users, which is an information-disclosure and bundle-size
regression. Always gate it behind `mode === 'development'` (or an equivalent
dev-only check) exactly as above.

---

## 4. Component conventions

- **Composition API with `<script setup>` only** — no Options API, no
  separate `<script>` block unless interop requires it.
- Import shared UI atoms from `frappe-ui` (`Button`, `Badge`, `Dialog`,
  `Alert`, `FormControl`, `TextInput`, `ErrorMessage`, `FeatherIcon`, ...)
  instead of re-implementing them; register any that need to be global in
  `main.js`'s `globalComponents` map, otherwise just import locally per
  component.
- `defineProps({...})` with explicit `type` and `required`/`default` for
  every prop; `defineEmits([...])` listing every emitted event name
  (kebab-case, e.g. `'change-credits'`, `'view-details'`).
- Extract repeated markup into small presentational components (e.g. a
  `DetailRow`, an `AvailabilityBadge`) rather than duplicating template
  blocks — but don't over-abstract single-use markup.
- Styling is Tailwind utility classes in the template; no `<style>` blocks.
  Reuse existing utility classes/patterns already defined in the app (e.g. a
  shared `card-surface` class) instead of inventing a parallel one.
- Path alias `@/` → `frontend/src` (set in `vite.config.js`'s
  `resolve.alias`) — use it for all intra-app imports instead of relative
  `../../` chains.
- Icons come from `~icons/lucide/<name>` (unplugin-icons via `lucideIcons`
  in the frappe-ui Vite plugin), imported per-icon, PascalCase
  (`LucideMinus`, `LucidePlus`).
- Wrap translatable UI strings in `__('...')` (the app's translation plugin,
  installed in `main.js`).
- Routes: never hardcode a route path/name string in a component — import
  from the app's `src/config/routes.js` (`ROUTE_NAMES`, `ROUTE_PATHS`,
  and any `portalUrl()`-style helper) so the base route stays a single
  source of truth shared with the backend's Python route constant.

---

## 5. Usage notes for Claude

1. Before changing `vite.config.js`, `main.js`, or anything touching
   `__VUE_PROD_DEVTOOLS__` / boot-data fetching, read the target app's
   actual files first — don't assume they match this file's `fms` example
   verbatim; confirm the plugin options, port, and route base in place.
2. Any devtools/debug-enabling change must be gated behind `mode ===
   'development'` (Vite) or `frappe.conf.developer_mode` (Python) — never
   unconditional, and never merged into the same code path the real
   production build/deploy uses.
3. When asked to add a page/route/store/component, follow the directory
   layout and conventions in sections 1 and 4 rather than inventing a new
   structure.
4. If a boot-data value needs to reach the frontend, it must be added
   symmetrically to both the `no_cache` `www` controller's `get_boot()` (or
   equivalent) and consumed the same way in `main.js`'s dev-mode
   `get_context_for_dev` branch — otherwise `yarn dev` and `yarn build` will
   disagree on what `window.*` contains.
5. If unsure whether the user is looking at the Vite dev server or the
   bench-served build, ask which URL/port they're using before diagnosing
   further Vue behavior differences (missing HMR, devtools, stale data,
   etc. are almost always explained by this).
