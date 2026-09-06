---
name: frappe-vue-js-override-and-new-page
description: >
  Use this skill when customizing a Frappe SPA frontend (Frappe CRM, HRMS, Gameplan,
  Helpdesk, Insights, or any frappe-ui + Vite + Vue 3 app) from a SEPARATE custom app,
  WITHOUT forking or editing the upstream repo. Triggers: "override the CRM sidebar",
  "add a button to Frappe CRM", "add a new page/route to Frappe CRM", "customize the
  HRMS frontend", "override AppSidebar.vue / a .vue component without forking", "render
  a video / embed / custom screen inside CRM", anything mentioning the `{custom_app}`
  app, an "overlay build", `assemble-frontend.mjs`, `overlay/<app>/mirror/`,
  `overlay/<app>/custom/`, or `@/custom/` imports. Covers both jobs: (A) shadowing an
  existing upstream .vue/.js file with an edited copy, and (B) adding a brand-new page
  wired into the router and linked from the nav/sidebar. Also use it to bootstrap the
  override app itself or to add a new target (e.g. HRMS after CRM).
tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
---

# Frappe Vue SPA — override & new page (no fork)

Customize a Frappe SPA (CRM today, HRMS tomorrow) from a **custom app**, keeping the
upstream repo untouched. The custom app commits only the changed/new source files plus
a small assembler script; every build regenerates a throwaway copy of the upstream
frontend with those files layered on top.

Reference app in this bench: **`{custom_app}`** at
`{bench}/apps/{custom_app}/`. Paths below assume `{bench}` = bench root. If a
`CLAUDE.md` exists it may override the app name / bench path — read it first.

Frappe SPA docs: https://docs.frappe.io/framework/user/en/portal-pages
frappe-ui Vite plugin: `frappe-ui/vite` (`frappeui({ buildConfig, jinjaBootData, lucideIcons })`)

---

## Mental model

1. **Overlay build, not a fork.** The custom app has no `frontend/` of its own in git.
   `scripts/assemble-frontend.mjs` copies `{bench}/apps/<target>/frontend` into a
   git-ignored scratch dir `{bench}/apps/<override_app>/.frontend-build/`, copies our
   `overlay/<target>/mirror/**` on top (same relative paths → they win), copies our
   `overlay/<target>/custom/**` into `src/custom/`, rewrites the build config so output
   lands in the custom app, then `yarn build` runs there.

2. **Serving is by www-template shadowing.** Frappe resolves `www/<name>.html`
   **last-installed-app-first** (`frappe.get_installed_apps()` reversed). If the custom
   app is installed *after* the target app, its `<override_app>/www/crm.html`
   (the built SPA entry) is served at `/crm` instead of the target's. A tiny
   `www/crm.py` re-exports the target's `get_context` so boot data is identical.

3. **Two kinds of source file:**

   | Kind | Lives in | Ends up at | How referenced |
   |---|---|---|---|
   | **Override** — edited copy of an upstream file | `overlay/<target>/mirror/<path>` (mirrors `apps/<target>/frontend/<path>` exactly) | shadows `<path>` in the build | same import paths as upstream |
   | **New file** — no upstream counterpart | `overlay/<target>/custom/<path>` | `src/custom/<path>` in the build | `import('@/custom/<path>')` |

---

## App layout

```
{bench}/apps/{custom_app}/
├── overlay/                        ← the ONLY tree you hand-edit
│   ├── README.md
│   ├── crm/                        ← one folder per upstream target app
│   │   ├── mirror/                 ← == apps/crm/frontend/  (paths match exactly)
│   │   │   └── src/…
│   │   └── custom/                 ← new files → src/custom/ → import @/custom/…
│   │       └── pages/Video.vue
│   └── hrms/                       ← same shape, filled when HRMS work starts
│       ├── mirror/
│       └── custom/
├── scripts/
│   └── assemble-frontend.mjs       ← copies upstream + overlay, repoints config
├── {custom_app}/                  ← python module
│   ├── hooks.py                    ← website_route_rules
│   ├── www/crm.py + crm.html       ← crm.py committed; crm.html is BUILT (git-ignored)
│   └── public/frontend/            ← BUILT assets (git-ignored)
├── .frontend-build/                ← scratch copy, git-ignored, regenerated each build
└── package.json                    ← type:module; assemble/build/dev scripts
```

`.frontend-build/` **must be a direct child of the app root** — the frappe-ui plugin
auto-derives `outDir` from `cwd/..` (looks for a sibling dir with `public/` + `hooks.py`),
and upstream config uses `../<target>/...` relative paths. A nested scratch dir breaks both.

---

## Task A — override an existing upstream .vue / .js file

**Example: add a "Watch Video" item to the CRM sidebar footer.**

1. **Find the upstream file** and take its path after `apps/<target>/frontend/`:
   ```
   apps/crm/frontend/ src/components/Layouts/AppSidebar.vue
   ```
   `grep -rn "text you see in the UI" {bench}/apps/crm/frontend/src` to locate it.

2. **Copy it verbatim** to the mirror path:
   ```
   overlay/crm/mirror/src/components/Layouts/AppSidebar.vue
   ```
   `mkdir -p` the dirs and `cp` the original — do not hand-type it.

3. **Add a header comment** recording the source path + reason, then make the smallest
   possible edit, marking every changed line `ours`:
   ```vue
   <!--
     OVERRIDE — Frappe CRM
     Shadows: apps/crm/frontend/src/components/Layouts/AppSidebar.vue
     Reason : add a "Watch Video" item to the sidebar footer (routes to /video).
     Only lines marked `ours` differ from upstream — re-sync on CRM upgrades.
   -->
   ```
   ```vue
   import VideoIcon from '~icons/lucide/video' // ours

   <!-- ours — routes to the /video page (see overlay/crm/custom/pages/Video.vue) -->
   <SidebarItem
     :label="__('Watch Video')"
     :to="{ name: 'Video' }"
     :active="activeItem === 'Video'"
     @click="selectItem($event, 'Video')"
   >
     <template #prefix><VideoIcon class="size-4 text-ink-gray-7" /></template>
   </SidebarItem>
   ```
   Icons: `~icons/lucide/<name>` (unplugin-icons virtual import — enabled via
   `lucideIcons: true`). `__()` is the global translation helper. `@/` = `src/`.

4. **Prefer overriding small, stable files.** To register a route, shadow `src/main.js`
   (short, rarely changes) and call `router.addRoute(...)` — **not** `src/router.js`
   (large, volatile, conflicts on every upstream change).

---

## Task B — add a brand-new page

**Example: a `/video` route that embeds a YouTube video.**

1. **Create the page** at `overlay/crm/custom/pages/Video.vue` (assembled to
   `src/custom/pages/Video.vue`). Use frappe-ui / Tailwind tokens
   (`bg-surface-white`, `text-ink-gray-9`, `border-outline-gray-1`, …) to match the
   host app. Header comment: mark it a NEW page and name the files that wire it in.
   ```vue
   <template>
     <div class="flex h-full flex-col overflow-hidden bg-surface-white">
       <header class="flex items-center gap-2 border-b border-outline-gray-1 px-5 py-3.5">
         <VideoIcon class="size-4 text-ink-gray-7" />
         <h1 class="text-lg font-semibold text-ink-gray-9">{{ __('Watch Video') }}</h1>
       </header>
       <div class="flex flex-1 items-center justify-center overflow-y-auto p-6">
         <div class="aspect-video w-full max-w-4xl overflow-hidden rounded-lg border border-outline-gray-2 bg-black">
           <iframe class="h-full w-full" :src="embedUrl" title="YouTube video player"
             frameborder="0" referrerpolicy="strict-origin-when-cross-origin"
             allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
             allowfullscreen />
         </div>
       </div>
     </div>
   </template>
   <script setup>
   import VideoIcon from '~icons/lucide/video'
   const YOUTUBE_ID = 'Oct6AbKtPZ8' // https://youtu.be/Oct6AbKtPZ8
   const embedUrl = `https://www.youtube-nocookie.com/embed/${YOUTUBE_ID}?rel=0`
   </script>
   ```

2. **Register the route** via a `src/main.js` override (Task A pattern). A static path
   outranks `router.js`'s `/:invalidpath` catch-all, so insertion order is irrelevant:
   ```js
   // ── ours ── page behind the sidebar's "Watch Video" button
   router.addRoute({
     path: '/video',
     name: 'Video',
     component: () => import('@/custom/pages/Video.vue'),
   })
   ```
   Add this right after the existing `import ... from './router'` / `App.vue` imports,
   before the `createApp` call.

3. **Link to it** from the sidebar/nav via a Task A override (see step 3 above:
   `:to="{ name: 'Video' }"`).

---

## The assembler — `scripts/assemble-frontend.mjs`

ESM (`package.json` has `"type": "module"`). For **one target** it:

1. `cp apps/<target>/frontend → .frontend-build/` (skips `node_modules`, `.git`, `dist`)
2. `cp overlay/<target>/mirror/** → .frontend-build/**`
3. `cp overlay/<target>/custom/** → .frontend-build/src/custom/**`
4. Rewrites the copied `package.json` + `vite.config.js`, replacing the target's names:
   | find | replace |
   |---|---|
   | `/assets/<target>/frontend/` | `/assets/<override_app>/<assetDir>/` |
   | `../<target>/public/frontend` | `../<override_app>/public/<assetDir>` |
   | `../<target>/www/<wwwName>.html` | `../<override_app>/www/<wwwName>.html` |
   | `"name": "<target>-ui"` | `"name": "<override_app>-<target>-ui"` |

A `.frontend-build/.target` marker holds the last target; switching targets wipes the
whole scratch dir (deps differ), otherwise `node_modules` is kept for a fast reinstall.

Targets are a config object in the script:
```js
const TARGETS = {
  crm:  { app: 'crm',  wwwName: 'crm',  assetDir: 'frontend' },
  hrms: { app: 'hrms', wwwName: 'hrms', assetDir: 'hrms' },
}
```
`assetDir` is `frontend` for CRM only because Frappe already symlinks
`sites/assets/<override_app>` → the app's `public/`; keep each target in its own subdir.

---

## Build & verify

```bash
cd {bench}/apps/{custom_app}

# assemble only (inspect the scratch dir)
node scripts/assemble-frontend.mjs crm

# full build — heap bump is REQUIRED, the CRM build OOMs at the default ~2 GB
NODE_OPTIONS=--max-old-space-size=8192 yarn build:crm     # or build:hrms
```

`package.json` scripts: `assemble`, `build` (→ `build:crm`), `build:crm`, `build:hrms`,
`dev:crm`, `dev:hrms`. Each does `node scripts/assemble-frontend.mjs <t> && cd
.frontend-build && yarn install --frozen-lockfile && yarn build`.

**Verify after a build:**
- `{custom_app}/public/frontend/assets/` contains a chunk for the new page (e.g. `Video-*.js`)
- `{custom_app}/www/crm.html` references `/assets/{custom_app}/frontend/` (not `/assets/crm/`)
- `{custom_app}/www/crm.html` still contains the `{% for key in boot %}` block (jinjaBootData)
- app is installed AFTER the target: `grep -n crm {bench}/sites/apps.txt` → `{custom_app}` below `crm`
- deploy: `bench build` is not needed (vite writes straight into `public/`); do
  `bench --site <site> clear-cache` and hard-refresh. In dev use `yarn dev:crm` +
  the frappe proxy.

---

## Bootstrapping the override app (or adding a target)

1. `bench new-app <override_app>` &&  `bench --site <site> install-app <override_app>`
   **after** the target app (order in `sites/apps.txt` decides www resolution).
2. Root `package.json`: `"private": true`, `"type": "module"`, the `assemble`/`build*`/`dev*`
   scripts above. No dependencies — the scratch dir installs the target's own.
3. `scripts/assemble-frontend.mjs` with a `TARGETS` entry per target app.
4. `<override_app>/hooks.py`:
   ```python
   website_route_rules = [
       {"from_route": "/crm/<path:app_path>", "to_route": "crm"},
   ]
   ```
   (The target usually registers this already; repeating it makes the app
   self-contained regardless of load order.)
5. `<override_app>/www/crm.py`:
   ```python
   from crm.www.crm import get_context  # noqa: F401
   no_cache = 1
   ```
   Plus an empty `<override_app>/www/__init__.py`.
6. `.gitignore`: `/.frontend-build/`, `/<override_app>/public/frontend/`,
   `/<override_app>/public/<assetDir>/`, `/<override_app>/www/crm.html`,
   `/<override_app>/www/hrms.html`.
7. `overlay/<target>/{mirror,custom}/` with a `.gitkeep` until populated.

For a **new target** (e.g. HRMS after CRM): add its `TARGETS` entry, create
`overlay/hrms/{mirror,custom}/`, add `www/hrms.py` (`from hrms.www.hrms import get_context`),
add the gitignore lines, then `yarn build:hrms`.

---

## Gotchas

- **Never edit `.frontend-build/`** — it is wiped/regenerated every build. Edit `overlay/`.
- **Heap:** the CRM frontend build OOMs (`FATAL ERROR: Reached heap limit`) without
  `NODE_OPTIONS=--max-old-space-size=8192`. Bake it into CI/deploy.
- **`main.js` over `router.js`** for adding routes — small stable file, no merge pain.
- **Scratch dir must be a direct child of the app root** (frappe-ui `outDir`
  auto-derivation + `../<target>/...` relative paths).
- **www resolution is install-order, not alphabetical** — the override app must be
  installed *after* its target or `/crm` still serves the upstream SPA.
- **Keep overrides byte-for-byte upstream except the `ours` lines** so they re-sync
  cleanly. Re-copy from upstream and re-apply the `ours` diff on target upgrades.
- **`@/custom/` is our convention**, resolved by the stock `@` → `src` alias
  (`src/custom/…`). No extra alias needed.
- **Don't commit** `.frontend-build/`, `public/frontend/`, or the built `www/crm.html`.
