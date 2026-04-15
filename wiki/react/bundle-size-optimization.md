# Bundle Size Optimization

> Sources: Nadia Makarevich, 2025
> Raw: [Web Performance Fundamentals](../../raw/react/web-performance-fundamentals-nadia-makarevich.md)

## Overview

Bundle size is one of the most impactful factors in web performance, affecting download times, JavaScript evaluation cost, and hydration delays in SSR applications. Even small oversights -- a wildcard import, a duplicated date library, a non-tree-shakable dependency -- can balloon a bundle from hundreds of kilobytes to multiple megabytes. This article covers how to measure, analyze, and reduce bundle size through code splitting, tree shaking, dependency auditing, and chunk preloading strategies.

## Why Bundle Size Matters

Large JavaScript bundles hurt performance in three distinct ways, each compounding the others.

### Network Impact (Download Time)

The most obvious cost: more bytes means longer downloads. A 5 MB uncompressed bundle on "Fast 4G" throttling produces an LCP of roughly 6 seconds versus 0.88 seconds for a 390 KB bundle. On Chrome's default 3G setting, a 5 MB bundle can take over 80 seconds to load.

**Compression helps enormously but can mask creeping bloat.** Enabling gzip or brotli on the server can compress a 5 MB bundle to around 1.1 MB transferred, dropping LCP from 6 seconds to 1.73 seconds on Fast 4G. However, this also means that adding a "small" 30 KB library may only add ~6 KB after compression -- negligible in isolation, but it compounds. Six months later, you wake up to 5 MB uncompressed and need a company-wide initiative to fix it.

Most CDN providers and hosting platforms enable compression by default. If yours does not, enable it immediately -- it is the single highest-impact, lowest-effort change for network performance.

> Reducing JavaScript size by 200 KB, while it seems impressive, could result in just a 40 KB decrease in the transferred size after compression. Which will have a much lesser impact (if any) on the initial load than one might hope for when they hear the "200 KB" number.

### JavaScript Evaluation Time (CPU Cost)

After download, the browser must **evaluate** (parse and compile) all the JavaScript before it can execute any of it. This is CPU-bound and blocks the main thread.

In a study project comparison with 6x CPU throttling:

| Bundle Size | Evaluate Task Duration |
|---|---|
| 390 KB | ~45 ms |
| 5,321 KB | ~740 ms |

That is a 16x increase for the evaluate step alone. The execution step (actually running React) stayed the same (~270 ms) because the page content was identical -- all the extra JavaScript was never used on that page.

**This cost is arguably worse than the network cost:** network costs are paid only on the first visit (subsequent visits use browser cache), but the evaluate task happens on every single page load, cached or not.

### Bundle Size and SSR

With SSR, the page becomes visible quickly (the HTML arrives fast), but it is not interactive until JavaScript downloads, evaluates, and hydrates. A 5 MB bundle on 4G creates a roughly 7-second gap where the page looks functional but nothing works -- buttons do not respond, toggles do nothing, and only native links and CSS animations function. During this gap, the page is just pure pre-rendered HTML with no JavaScript attached. The page seems very fast, but in reality it is broken for several seconds.

This gap between visibility and interactivity is measured by **Time To Interactive (TTI)**. It was removed from Lighthouse and Core Web Vitals, but remains a useful metric for analyzing SSR performance. The TTI metric specifically captures what the user experiences: the page appears loaded, they try to click a button or toggle a setting, and nothing happens. On slower connections (e.g., 3G), the gap can extend to tens of seconds.

The combined effect of slow network + slow CPU makes large bundle sizes particularly problematic for SSR. Users see the page, try to interact, and conclude the site is broken. This is arguably worse than a slow CSR load where the user at least sees a loading indicator -- with SSR and large bundles, there is no visual indication that the page is not yet ready.

## Reducing Bundle Size with Code Splitting

Code splitting breaks a monolithic bundle into smaller "chunks" that can be downloaded in parallel. The bundler traces the dependency graph and slices it according to rules you configure.

### Vendor Chunks

The first and most important split: separate your application code from third-party dependencies.

```js
// vite.config.ts (Rollup manualChunks)
manualChunks: (id) => {
  if (id.includes('node_modules')) {
    return 'vendor';
  }
  return null;
};
```

**Why this matters beyond parallelism:** Modern bundlers generate content-hashed filenames (e.g., `vendor-DIQ9qPEN.js`). The hash only changes when file content changes. If all code is in one file, every deployment invalidates the cache for everything -- including React itself, which has not changed. Splitting vendors means the vendor chunk's hash stays stable across app code deployments, and returning visitors only re-download the (typically small) application chunk.

Measurement in the study project showed that just adding a vendor chunk cut the JavaScript compile time in half.

### Granular Chunking Strategies

You can go further and split by feature, by page, or by heavy library:

```js
manualChunks: (id) => {
  if (id.includes('node_modules')) {
    if (id.includes('@radix')) {
      return 'radix';
    }
    return 'vendor';
  }
  if (id.includes('pages/inbox')) {
    return 'inbox';
  }
  if (id.includes('pages/dashboard')) {
    return 'dashboard';
  }
  if (id.includes('frontend/components')) {
    return 'components';
  }
  return null;
};
```

Developing a good chunking strategy is largely trial and error, depending on the number of external dependencies and code structure. A good target chunk size is around **50-100 KB** -- anything larger needs further splitting.

### Chunks Preloading

All manual chunks (regardless of the tool) are preloaded as critical resources. In Vite, every chunk except the default `index` ends up in the `<head>` as `<link rel="modulepreload" />`:

```html
<script type="module" crossorigin src="/assets/index-pLa1GZqS.js"></script>
<link rel="modulepreload" crossorigin href="/assets/vendor-DIQ9qPEN.js" />
```

This allows the browser to fetch and compile all JavaScript in parallel, which is good. However, because we did not indicate which chunks are non-critical, **the browser waits for all of them** before rendering. Manual chunks alone do not defer non-critical code -- for that, you need lazy loading with dynamic imports.

### HTTP/1 vs HTTP/2 and Code Splitting

On HTTP/1, Chrome limits concurrent connections to **six per domain**. If you create more than six chunks (e.g., eight chunks total), the 7th and 8th downloads are queued. Worse, the CSS file can also be delayed -- so if the page was SSRed, splitting JavaScript into too many chunks would delay the initial paint and make performance worse.

For example, a granular chunking strategy like this produces eight chunks:

```js
manualChunks: (id) => {
  if (id.includes('node_modules')) {
    return 'vendor';
  }
  if (id.includes('frontend/components')) {
    return 'components';
  }
  if (id.includes('pages/inbox')) {
    return 'inbox';
  }
  if (id.includes('pages/dashboard')) {
    return 'dashboard';
  }
  if (id.includes('pages/login')) {
    return 'login';
  }
  if (id.includes('pages/settings')) {
    return 'settings';
  }
  if (id.includes('frontend/patterns')) {
    return 'patterns';
  }
  return null;
};
```

On HTTP/1, only six of these chunks download initially. The remaining chunks (and critically, the CSS file) are queued until one of the first six completes. If the page was server-side rendered, this delay to the CSS file would delay the initial paint and make performance numbers worse than having fewer chunks.

**HTTP/2 and HTTP/3 handle many more requests in parallel** through multiplexing. As of 2022-2023, roughly 90% of CDN traffic uses HTTP/2+. So production websites behind a CDN are typically fine.

However, **local development servers typically use HTTP/1** (Vite's preview server, Express, and even Next.js dev mode). HTTP/2 needs to be implemented separately for local environments. This creates a dangerous trap: if you experiment with chunking strategies locally, you may see zero gains (or regressions) that do not reflect production behavior. Or the opposite -- local results may look bad while production would show improvement. Always be aware of which protocol your measurement environment is using before drawing conclusions about chunking strategies.

### Chunk Size and Compression

There is a second downside to splitting into too many small chunks: **compression ratio decreases** as chunks become smaller. Compression algorithms like gzip and brotli work by finding repeated patterns in the data -- larger files provide more opportunities for pattern matching and therefore compress more efficiently. When you split a large bundle into many small chunks, each chunk compresses less effectively on its own. The overall network transfer of compressed JavaScript may therefore **increase**, even if the total uncompressed size stays the same or decreases. This trade-off especially matters for users on limited bandwidth and mobile devices, where every additional kilobyte of transfer has a measurable impact on load time.

## Analyzing Bundle Size

Before optimizing, you need to see what is in the bundle. Use a bundle analyzer tool appropriate for your bundler:

| Bundler | Tool |
|---|---|
| Vite/Rollup | `rollup-plugin-visualizer` |
| Webpack | `webpack-bundle-analyzer` |
| Any (source maps) | `source-map-explorer` |

```js
// vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer';

plugins: [
  visualizer({
    filename: 'stats.html',
    emitFile: true,
    template: 'treemap', // or 'flamegraph'
  }),
];
```

After building, open `dist/client/stats.html` in a browser. The visualization is a hierarchical map where block size corresponds to code size relative to the total. Click on blocks to zoom in.

### The Investigation Process

After generating the visualization, the work becomes a structured investigation. Apply the following process for **every** suspicious package you discover -- it is a repeatable methodology, not a one-time exercise:

**Step 1: Identify a package to eliminate.** Find unreasonably large blocks (usually in `node_modules`). The npm naming convention helps: packages are either one word with dashes, or `@namespace/package-name`. Everything directly under `node_modules` in the visualization is either a package or a namespace (starting with `@`) containing multiple packages.

**Step 2: Understand the package.** Google the package to learn what it does. Understanding what it provides is essential before deciding whether it can be removed or replaced.

**Step 3: Understand its usage in the codebase.** Search your codebase for direct imports of the package name. If you find direct imports, analyze the code to understand how and why it is used -- sometimes the usage is legitimate, sometimes it is a leftover from incomplete refactoring. If no direct imports exist, it is a transitive dependency pulled in by another library (use `npm-why` to trace the chain).

**Step 4: Confirm it is the problem before refactoring.** Before attempting any refactoring -- which in real-world projects could be very costly -- confirm that you have identified the problem correctly. Comment out the import, rebuild, and verify the bundle shrinks. The build will likely fail (since the usage is still in the code), but the produced bundle size will reveal whether the import was truly the source of the bloat. Do not skip this step. In the study project, commenting out two `@mui` imports dropped the vendor from 5 MB to 811 KB -- confirmation that those imports were the problem, not something else.

**Step 5: Fix** -- Apply the appropriate fix (targeted imports, library replacement, or removal). Then verify the app still works.

## Tree Shaking and Dead Code Elimination

Modern bundlers build a dependency tree from all imports/exports and remove "dead branches" -- code that is exported but never imported by anything in the application. This is called tree shaking.

### How It Works

```js
// This button is exported but never imported anywhere:
export const MyButton = () => <button>Click me</button>;
// After build, MyButton's code is completely absent from the bundle.
```

Tree shaking follows the import chain transitively. If `MyDialog` imports `MyButton`, but `MyDialog` itself is never rendered in the application tree, both are eliminated. Only when a component is actually used in code that forms the running app does it appear in the final bundle.

Modern bundlers are getting smarter and smarter, and it becomes harder to fool them. But it is still possible.

### What Breaks Tree Shaking: `import *` with Re-export

Even with ESM-compatible libraries where tree shaking normally works, the **`*` import combined with assignment to a variable/object** defeats tree shaking:

```js
// This pattern prevents tree shaking!
import * as Material from '@mui/material';

export const StudyUi = {
  Library: Material,  // Bundler cannot determine which parts are used
};
```

When `Material` is used as a variable rather than just a means to extract named exports, the bundler cannot statically determine which parts are actually referenced. It includes everything.

In the study project, this pattern inflated the vendor from 811 KB to over 5 MB by pulling in every MUI component and all 2,000 MUI icons. The fix is to import only what you use:

```js
import { Snackbar } from '@mui/material';
import { Star } from '@mui/icons-material';

export const StudyUi = {
  Library: { Snackbar },
};
export const Icons = {
  Star,
};
```

After this fix, the bundle dropped to 878 KB. Note that the `*` import by itself (used only to extract named exports directly) does not break tree shaking -- the bundler is smart enough for that. It is the re-assignment to an object property that causes the problem.

### Why This Pattern Exists

Namespacing through `import *` is popular because it simplifies imports and provides clarity:

```js
import { Ui } from '@fe/components';

// Instead of importing dozens of individual components:
<Ui.Dialogs.Dialog />
<Ui.Buttons.SmallButton />
```

For your own code, this may not matter much -- you probably use most of what you write. But for external libraries with hundreds of exports, it is devastating.

## ES Modules and Non-tree-shakable Libraries

Tree shaking only works reliably with **ES Modules** (ESM) -- the `import`/`export` syntax. JavaScript has several module formats -- ESM, CJS (CommonJS), AMD, UMD -- each defining how reusable code is loaded into other code. When you see `import { bla } from 'bla-bla'` or `export const bla` or `export { bla }`, that is ESM format. ESM is the modern standard for frontend code, and modern bundlers can tree-shake it reliably. Everything else (CommonJS `require()`, AMD, UMD) is very difficult or impossible to tree-shake because the bundler cannot statically analyze dynamic `require()` calls the way it can analyze static `import` declarations.

### Verifying Tree Shaking Failure

When you suspect tree shaking is not working for a library, verify it experimentally. For example, if you use two functions from a library, remove one and rebuild. If tree shaking works, the bundle size should decrease and the chunk name should change. If nothing changes, tree shaking has failed and the entire library is being included regardless of what you import.

You can check whether a library uses ESM with the `is-esm` CLI tool:

```bash
npx is-esm lodash          # No  -- not tree-shakable
npx is-esm @mui/material   # Yes -- tree-shakable
```

When a non-ESM library like lodash is imported, the **entire library** ends up in the bundle regardless of how you import it:

```js
// Both of these include ALL of lodash:
import _ from 'lodash';
import { trim } from 'lodash';
```

Changing from `import _` to `import { trim }` changed the bundle size by exactly two bytes -- just the renamed minified variable.

**Workaround: targeted sub-path imports.** Many libraries provide per-function entry points even if the main entry is not ESM:

```js
// Only includes the trim and lowerCase functions:
import trim from 'lodash/trim';
import lowerCase from 'lodash/lowerCase';
```

This reduced the vendor chunk from 878 KB to 813 KB in the study project. Better yet, replace with native equivalents when possible:

```js
// Native JavaScript -- zero bundle cost:
const cleanValue = val.toLowerCase().trim();
```

## Common Sense and Repeating Libraries

In large projects with multiple teams, it is common to accumulate **duplicate libraries that solve the same problem**. This is especially frequent with functionality that is too painful to implement from scratch and generic enough to extract into a library: dates, animations, resizing, infinite scrolling, forms, charts, and styling. Different teams (or even the same team at different points in time) independently add libraries without checking what already exists.

The study project contained three date libraries:

- `date-fns` -- tree-shakable, modern
- `moment` -- not tree-shakable, no targeted imports, large
- `luxon` -- tree-shakable but naturally larger than date-fns

Each was used in exactly one place for the same purpose: formatting a date to a human-readable string.

### Identification and Consolidation Strategy

The decision of which library to keep depends on how much code would need to be refactored, how much effort it takes, and how many kilobytes of bundle size you are willing to tolerate. In old projects, one library (e.g., Moment) may be used everywhere while newer alternatives are only in a few places -- making it more practical to remove the newer ones as a quick win. Or the opposite: the old library may be a leftover from a large refactoring that someone forgot to clean up.

When you can choose freely, the investigation process is:

1. Check which libraries are tree-shakable (`moment` is not, so it is the worst offender).
2. Compare compressed sizes in the bundle visualization. Even among tree-shakable libraries, some are naturally larger.
3. Consider the API quality, maintenance status, and community support.
4. Pick one and refactor. Removing `moment` and `luxon` in favor of `date-fns` reduced the vendor by 20% (804 KB to 672 KB).

Other common areas for duplication: animation libraries, CSS-in-JS solutions (e.g., having both Tailwind and Emotion), form validation, charting, infinite scrolling, and floating/positioning libraries.

Also watch for libraries that exist due to incomplete refactoring. The study project had `@emotion/styled` used in exactly one place -- a leftover from a migration to Tailwind. A single `styled.div` with `text-align: center` was pulling in the entire Emotion runtime. Replacing it with a Tailwind className eliminated the dependency.

## Transitive Dependencies

A library that does not appear in your direct code may still be in the bundle because another dependency requires it. These are called transitive dependencies. They often show up with "foundation" level libraries -- positioning utilities, CSS-in-JS runtimes, and general-purpose utilities like lodash -- that other libraries build on top of.

The `npm-why` CLI tool traces the full dependency chain to reveal where a package originates. This is essential when removing a direct usage of a library does not reduce the bundle, because a transitive consumer is still pulling it in.

In the study project, removing direct usage of `@emotion/styled` did not shrink the bundle because `@mui/material` depends on it transitively. Running `npm-why` revealed the full chain:

```bash
npx npm-why @emotion/styled
# Output:
# study-project > @emotion/styled@11.14.0
# study-project > @mui/material > @emotion/styled@11.14.0
# study-project > @mui/material > @mui/system > @emotion/styled@11.14.0
# study-project > @mui/material > @mui/system > @mui/styled-engine > @emotion/styled@11.14.0
```

The only way to remove a transitive dependency is to remove **everything** that depends on it. This can escalate a "quick fix" into a significant refactoring effort. In the study project, removing both `@mui` packages entirely (replacing Snackbar with Radix Toast, MUI Star icon with a local SVG icon) eliminated `@emotion` and dropped the vendor chunk by another ~70 KB to 600 KB.

This is always a trade-off between the time spent on refactoring and the potential benefits. In real code, the decision depends on the scale of usage throughout the application.

## Key Takeaways

- **Compression is essential** but masks incremental bloat -- track uncompressed sizes too.
- **JavaScript evaluation cost** is paid on every visit, not just the first. This often matters more than download time.
- **SSR amplifies bundle size problems** by creating a visible-but-broken gap before hydration.
- **Vendor chunks** should be your first code-splitting step -- they improve both parallelism and cache hit rates.
- **Tree shaking requires ES Modules** and breaks when you use `import *` combined with object re-exports.
- **Always verify with a bundle analyzer** before and after optimization. Do not guess -- measure.
- **Check for duplicate libraries** solving the same problem (dates, animations, UI components).
- **Transitive dependencies** can be the real bloat source. Use `npm-why` to trace them.
- **HTTP/2 removes the 6-connection limit**, making aggressive code splitting viable in production, but local dev servers may still use HTTP/1.
- A good chunk size target is **50-100 KB**; smaller chunks lose compression efficiency.

## See Also

- [Lazy Loading and Suspense](lazy-loading-and-suspense.md)
- [React Server Components](react-server-components.md)
- [Web Performance Metrics](web-performance-metrics.md)
- [Rendering Strategies](rendering-strategies.md)
