---
name: vite-chunk-optimizer
description: >
  Diagnose and fix slow page loads in Vite projects by analyzing and optimizing chunk splitting.
  Use this skill whenever a user says a page is slow to load, takes too long to open, has a large
  bundle, or wants to optimize Vite build performance. Also trigger when users mention chunk size,
  code splitting, lazy loading routes, or bundle optimization in a Vite/Rolldown context — even if
  they don't say "chunk" explicitly. Works with any Vite framework (Vue, React, Svelte, Solid, etc.).
---

# Vite Chunk Optimizer

You help users fix slow-loading pages in Vite projects by diagnosing which chunks are too large and applying targeted optimizations.

## Why pages load slowly in Vite apps

A slow page usually means the browser is downloading and parsing more JavaScript than that page actually needs. Common causes:

- **No route-level code splitting** — all pages bundled into one giant file
- **A single vendor chunk** containing every dependency, even ones only used by one page
- **Eagerly imported heavy libraries** (chart libs, editors, PDF viewers) that should be loaded on demand

## Workflow

Follow these steps in order. Each step builds on the previous one — don't skip the diagnosis phase, because applying optimizations blindly can make things worse (e.g., creating too many small chunks adds HTTP overhead).

**Important:** Before executing each step, print a log message to the user in the format: `[Step N/8] <description>` — e.g., `[Step 1/8] Understanding which page is slow...`. This keeps the user informed of progress.

### Step 1: Understand which page is slow

Ask the user which page or route feels slow if they haven't said. Find out:

- Is it the initial load (first paint) or a navigation (route change)?
- Does it happen in dev mode, production, or both?

Dev mode slowness is usually Vite's on-demand transform pipeline (different problem). This skill focuses on **production build** chunk optimization.

### Step 2: Check for Chrome coverage data

Ask the user: **Do you have a Chrome DevTools coverage JSON export from a production build? If so, please provide the file path.**

**STOP HERE. Do NOT proceed to any subsequent steps, do NOT launch parallel agents, do NOT read config files or run builds. End your turn after asking this question and wait for the user's reply.**

- If the user says no or skips: proceed to Step 3.
- If the user provides a file path **OR you discover a coverage JSON file in the repo** (e.g., via git status, glob): proceed to **Step 2b** below.

### Step 2b: Sourcemap + coverage analysis

**BLOCKING GATE: When coverage data is available (user-provided or discovered in the repo), you MUST complete Step 2b in full before starting Steps 3, 4, 5, 6, or 7. Do NOT launch parallel agents or read other files to "get ahead" — the whole point is that 2b's output replaces guesswork. Any optimization applied without completing 2b first is guessing, not diagnosing.**

**Why this matters:** Chunk-level coverage (which chunks loaded, what % was used) is NOT enough. A single chunk can contain dozens of modules. Without sourcemaps, you can only see "this 800 KB chunk was 20% used on the Login page" — but you can't tell WHICH modules inside it are evaluated vs dead weight. With sourcemaps, you can map coverage byte-ranges back to **individual source modules** and answer:

- Which specific modules inside a shared chunk are actually **evaluated** on the target page?
- Which modules are just along for the ride because they share a chunk with something needed?
- Which store/utility/library modules does the page actually execute vs merely download?

This is the difference between guessing from the import graph and **knowing exactly** what the browser evaluates.

**Steps:**

1. **Enable sourcemaps** — Verify `build.sourcemap` is set to `true` in `vite.config.ts` (or `.js`, `.mjs`). If not, enable it:
    
    ```tsx
    build: {
      sourcemap: true,
    }
    ```
    
2. **Rebuild with sourcemaps** — Run `npx vite build`. This generates `.js.map` files alongside each chunk.
3. **Have the user re-collect coverage** — The coverage JSON must be collected from a build that has sourcemaps. If the existing coverage was collected from a build without sourcemaps, ask the user to:
    - Run `npx vite preview` (or serve the `dist/` folder)
    - Open Chrome DevTools → Sources → Coverage → Start → Load the slow page → Stop → Export JSON
4. **Map coverage to source modules** — For each JS entry in the coverage JSON:
    - The `url` field identifies the chunk (e.g., `style.abc123.js`)
    - The `ranges` array contains byte offsets of code that was **actually evaluated**
    - Load the corresponding `.js.map` sourcemap file
    - Use the sourcemap to translate each coverage range's byte offsets back to original source file paths and line numbers
    - This tells you exactly which source modules were evaluated
5. **Identify optimization targets** — From the mapped data, look for:
    - Large modules that are **in the chunk but never evaluated** on the target page → candidates for lazy loading or splitting into a separate chunk
    - Modules from `node_modules` that are evaluated but only needed by other pages → candidates for `manualChunks` or deferred imports
    - Store/utility modules that are pulled into a shared chunk but only used post-login → candidates for lazy initialization

**This analysis directly informs Steps 6-7.** The fixes you apply should target the specific modules identified here, not just chunk-level heuristics.

**After completing Step 2b:** proceed to Step 3. Steps 3-5 provide supplementary context (config, routes), but Step 2b's module-level data is the primary signal for deciding what to optimize. Do not contradict or ignore 2b's findings based on import-graph guesses from Steps 3-5.

### Step 3: Analyze the current build output

run a production build:

```bash
npx vite build
```

Read the generated markdown report. Look for:

- Not all chunks over **200 KB** are optimization targets — only focus on those reachable from the target page's entry chunk
- The chunk names tell you what's in them (vendor, index, page names)

### Step 4: Read the Vite config

Read `vite.config.ts` (or `.js`, `.mjs`) and look for:

- `build.rolldownOptions.output.manualChunks` — existing chunk splitting rules
- `build.chunkSizeWarningLimit` — if raised, someone was hiding warnings instead of fixing them
- Any plugins that affect bundling

### Step 5: Check route definitions for lazy loading

Find the router config (common locations below) and check whether routes use dynamic imports:

| Framework | Typical location | Lazy pattern |
| --- | --- | --- |
| Vue Router | `src/router/index.ts` | `component: () => import('./views/Page.vue')` |
| React Router | `src/routes.tsx` or `src/App.tsx` | `lazy(() => import('./pages/Page'))` |
| SvelteKit | `src/routes/` | File-based (automatic) |
| Solid | `src/routes.ts` | `lazy(() => import('./pages/Page'))` |

Routes that use **static imports** (`import Page from './Page'`) get bundled into the entry chunk — this is the most common cause of slow initial loads.

### Step 6: Identify the specific problem

Based on Steps 2b-5, classify the issue. If you have sourcemap + coverage data from Step 2b, use the **module-level** evaluation data as your primary signal — it tells you exactly which modules are loaded but not needed on the target page. If you only have chunk-level data, use the import graph analysis from Steps 3-5 as a fallback, but be aware this is less precise:

**Problem A: Routes not lazy-loaded**
→ Convert static route imports to dynamic imports. This is the highest-impact fix.

**Problem B: One huge vendor chunk**
→ Split vendors by usage pattern using `manualChunks`. Group libraries by which pages actually use them.

**Problem C: A heavy library loaded eagerly on one page**
→ Dynamic-import the library at the component level, not just the route level. For example, a chart library should only load when the chart component mounts.

### Step 7: Apply targeted fixes

Apply only the fixes that match the diagnosed problems. Below are patterns for each.

### Fix A: Lazy-load routes

**Vue Router example:**

```tsx
// Before — static import, bundled into main chunk
import Dashboard from '@/views/Dashboard.vue'

// After — dynamic import, gets its own chunk
const Dashboard = () => import('@/views/Dashboard.vue')
```

**React Router example:**

```tsx
// Before
import Dashboard from './pages/Dashboard'

// After
import { lazy } from 'react'
const Dashboard = lazy(() => import('./pages/Dashboard'))
// Wrap in <Suspense> at the router level
```

### Fix B: Split vendor chunks by usage

In `vite.config.ts`, add `manualChunks` to group dependencies logically:

```tsx
build: {
  rolldownOptions: {
    output: {
      manualChunks(id) {
        if (id.includes('node_modules')) {
          // Core framework — needed everywhere, cache long-term
          if (id.includes('vue') || id.includes('react') || id.includes('svelte')) {
            return 'vendor-framework'
          }
          // Identify heavy libs from the build analysis and split them out
          // Only split libs that are large (>50KB) and used by few pages
          // Example patterns — adapt based on what the build analysis revealed:
          if (id.includes('echarts') || id.includes('chart.js')) {
            return 'vendor-charts'
          }
          if (id.includes('monaco-editor') || id.includes('codemirror')) {
            return 'vendor-editor'
          }
          if (id.includes('xlsx') || id.includes('pdfjs') || id.includes('mammoth')) {
            return 'vendor-docs'
          }
          // Everything else stays in a shared vendor chunk
          return 'vendor-common'
        }
      }
    }
  }
}
```

The specific split categories should come from Step 2's analysis — don't guess, look at what's actually large in the bundle.

### Fix C: Dynamic-import heavy components

For components that use heavy libraries, load the library on demand:

```tsx
// Vue example with defineAsyncComponent
import { defineAsyncComponent } from 'vue'

const HeavyChart = defineAsyncComponent(() => import('./components/HeavyChart.vue'))
```

```tsx
// React example
const HeavyChart = lazy(() => import('./components/HeavyChart'))
```

### Step 8: Verify the improvement

After applying fixes, rebuild and compare:

```bash
npx vite build 2>&1 | tail -60
```

Compare chunk sizes before and after. The target page's relevant chunks should be noticeably smaller. If the user wants a visual comparison, run the bundle visualizer again.

## Important caveats

- **Don't over-split.** Too many tiny chunks (< 20 KB) add HTTP request overhead that can make things slower. The sweet spot is chunks between 20-200 KB gzipped.
- **Don't blindly copy manualChunks recipes from the internet.** The right split depends on the project's actual dependency graph. Always analyze first.
- **Dynamic imports add a loading delay.** For above-the-fold content, prefetching or preloading the chunk is better than lazy-loading it. Vite automatically adds `<link rel="modulepreload">` for direct imports, but dynamically imported chunks need manual preload hints if they're critical.
- **Dev mode is different.** Vite's dev server uses native ESM and doesn't bundle — chunk splitting only affects production builds. If the user's problem is dev mode performance, this skill doesn't apply; look into Vite's `optimizeDeps` and server warmup instead.
