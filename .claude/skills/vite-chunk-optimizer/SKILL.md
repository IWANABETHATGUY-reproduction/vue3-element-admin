---
name: vite-chunk-optimizer
description: >
  Diagnose and fix slow page loads in Vite projects by analyzing and optimizing chunk splitting.
  Use this skill whenever a user says a page is slow to load, takes too long to open, has a large
  bundle, or wants to optimize Vite build performance. Also trigger when users mention chunk size,
  code splitting, lazy loading routes, or bundle optimization in a Vite/Rollup context — even if
  they don't say "chunk" explicitly. Works with any Vite framework (Vue, React, Svelte, Solid, etc.).
---

# Vite Chunk Optimizer

You help users fix slow-loading pages in Vite projects by diagnosing which chunks are too large and applying targeted optimizations.

## Why pages load slowly in Vite apps

A slow page usually means the browser is downloading and parsing more JavaScript than that page actually needs. Common causes:

- **No route-level code splitting** — all pages bundled into one giant file
- **A single vendor chunk** containing every dependency, even ones only used by one page
- **Eagerly imported heavy libraries** (chart libs, editors, PDF viewers) that should be loaded on demand
- **Barrel files** (`index.ts` re-exporting everything) that defeat tree-shaking
- **Duplicated modules** appearing in multiple chunks because Rollup doesn't know they're shared

## Workflow

Follow these steps in order. Each step builds on the previous one — don't skip the diagnosis phase, because applying optimizations blindly can make things worse (e.g., creating too many small chunks adds HTTP overhead).

### Step 1: Understand which page is slow

Ask the user which page or route feels slow if they haven't said. Find out:
- Is it the initial load (first paint) or a navigation (route change)?
- Does it happen in dev mode, production, or both?

Dev mode slowness is usually Vite's on-demand transform pipeline (different problem). This skill focuses on **production build** chunk optimization.

### Step 2: Analyze the current build output

Run a production build and examine the output:

```bash
npx vite build 2>&1 | tail -60
```

Vite logs every chunk with its size. Look for:
- Any chunk over **200 KB** (gzipped) — these are optimization targets
- The chunk names tell you what's in them (vendor, index, page names)

If the project has a `stats.html` or uses `rollup-plugin-visualizer`, open that instead — it gives a treemap of exactly what's inside each chunk.

If neither exists, temporarily add the visualizer to get a clear picture:

```bash
npx vite-bundle-visualizer
```

This generates a treemap HTML file without modifying the project config.

### Step 3: Read the Vite config

Read `vite.config.ts` (or `.js`, `.mjs`) and look for:
- `build.rollupOptions.output.manualChunks` — existing chunk splitting rules
- `build.chunkSizeWarningLimit` — if raised, someone was hiding warnings instead of fixing them
- Any plugins that affect bundling

### Step 4: Check route definitions for lazy loading

Find the router config (common locations below) and check whether routes use dynamic imports:

| Framework | Typical location | Lazy pattern |
|-----------|-----------------|--------------|
| Vue Router | `src/router/index.ts` | `component: () => import('./views/Page.vue')` |
| React Router | `src/routes.tsx` or `src/App.tsx` | `lazy(() => import('./pages/Page'))` |
| SvelteKit | `src/routes/` | File-based (automatic) |
| Solid | `src/routes.ts` | `lazy(() => import('./pages/Page'))` |

Routes that use **static imports** (`import Page from './Page'`) get bundled into the entry chunk — this is the most common cause of slow initial loads.

### Step 5: Identify the specific problem

Based on Steps 2-4, classify the issue:

**Problem A: Routes not lazy-loaded**
→ Convert static route imports to dynamic imports. This is the highest-impact fix.

**Problem B: One huge vendor chunk**
→ Split vendors by usage pattern using `manualChunks`. Group libraries by which pages actually use them.

**Problem C: A heavy library loaded eagerly on one page**
→ Dynamic-import the library at the component level, not just the route level. For example, a chart library should only load when the chart component mounts.

**Problem D: Duplicated modules across chunks**
→ Extract shared modules into a common chunk so they're downloaded once.

**Problem E: Barrel file pulling in everything**
→ Replace barrel imports with direct file imports.

### Step 6: Apply targeted fixes

Apply only the fixes that match the diagnosed problems. Below are patterns for each.

#### Fix A: Lazy-load routes

**Vue Router example:**
```ts
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

#### Fix B: Split vendor chunks by usage

In `vite.config.ts`, add `manualChunks` to group dependencies logically:

```ts
build: {
  rollupOptions: {
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

#### Fix C: Dynamic-import heavy components

For components that use heavy libraries, load the library on demand:

```ts
// Vue example with defineAsyncComponent
import { defineAsyncComponent } from 'vue'

const HeavyChart = defineAsyncComponent(() => import('./components/HeavyChart.vue'))
```

```tsx
// React example
const HeavyChart = lazy(() => import('./components/HeavyChart'))
```

#### Fix D: Extract shared chunks

If the build shows the same module in multiple chunks, configure a minimum shared threshold:

```ts
build: {
  rollupOptions: {
    output: {
      manualChunks(id) {
        // ... existing rules
      }
    }
  },
}
```

### Step 7: Verify the improvement

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
