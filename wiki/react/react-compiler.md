# React Compiler

> Sources: Nadia Makarevich, 2025
> Raw: [Web Performance Fundamentals](../../raw/react/web-performance-fundamentals-nadia-makarevich.md)

## Overview

The React Compiler is a build-time tool, implemented as a Babel plugin, that automatically adds memoization to React components. It eliminates the need for developers to manually apply `useMemo`, `useCallback`, and `React.memo` -- patterns that are notoriously fragile and error-prone when done by hand. The Compiler is **not** part of React itself; it is a separate tool that plugs into the build system. It supports React 17 and above, and was in Release Candidate status as of April 2025, with frameworks like Next.js still marking it as "experimental" in June 2025. While the Compiler can dramatically improve [interaction performance](interaction-performance.md) by eliminating unnecessary [re-renders](react-re-renders.md) -- in some cases reducing INP by more than 10x -- it is not a silver bullet. It cannot catch every re-render, it does not optimize external libraries, and overly complex component code can defeat its analysis. The trade-off is more "magic" in how React code behaves, making it harder to predict re-render behavior by reading the source alone.

## What It Is (and Is Not)

Many people assume that React Compiler is part of React. It is not. It is a separate tool implemented as a **Babel plugin** -- part of the build system that transforms the code developers write into something the browser understands.

This distinction has practical consequences:

- **Updating React alone does not enable it.** You must install the Compiler separately.
- **It requires a build system that supports Babel plugins.** Not all build tools are compatible. The official React docs list supported tools.
- **It works with older React versions.** React 17 is the minimum supported version, so even teams stuck on older releases can benefit.

### Timeline

| Date | Milestone |
|------|-----------|
| December 2021 | Introduced as a concept |
| October 2024 | Released as beta |
| April 2025 | Released as Release Candidate |
| June 2025 | Still marked "experimental" in Next.js |

Setting it up is relatively straightforward on compatible systems -- install the plugin, include it in the configuration, and it works. In Next.js, it is a matter of toggling a flag. The harder part is cleaning up the codebase to prepare for the Compiler. To help with that, the React team released an **ESLint plugin** that detects code patterns the Compiler cannot optimize. Install it and fix everything it finds before enabling the Compiler.

## What the Compiler Does

Like any Babel plugin, the Compiler transforms your source code at build time. Its job is to automatically apply [memoization](memoization.md) -- the same memoization that developers would otherwise need to do manually with `useMemo`, `useCallback`, and `React.memo`.

### The Manual Memoization Problem

Consider this component:

```jsx
const App = () => {
  return (
    <HeavyComponent
      arrayProp={[1, 2, 3]}
      callbackProp={() => {
        console.log('Callback happened');
      }}
    >
      <ChildComponent />
    </HeavyComponent>
  );
};
```

To prevent `HeavyComponent` from [re-rendering](react-re-renders.md) unnecessarily, a developer must manually memoize in **four** different places: the component itself (via `React.memo`), the array prop, the callback prop, and the children element. The fully memoized version looks like this:

```jsx
const App = () => {
  const memoArray = useMemo(() => [1, 2, 3], []);
  const memoCallback = useCallback(() => {
    console.log('Callback happened');
  }, []);
  const memoChild = useMemo(() => <ChildComponent />, []);

  return (
    <HeavyComponent arrayProp={memoArray} callbackProp={memoCallback}>
      {memoChild}
    </HeavyComponent>
  );
};
```

This is tedious, fragile, and has more chances to go wrong than right. Miss a single dependency or forget one wrapper, and the entire [memoization chain breaks](memoization.md).

### How the Compiler Transforms Code

The Compiler takes the first (non-memoized) version and transforms it into code that **behaves** like the second (memoized) version. However, it does not literally wrap things in `useMemo` and `useCallback`. It is smarter: it predicts dependencies, leverages conditional rendering, and groups related elements together.

Here is the actual output the Compiler produces for the simple `App` component:

```jsx
const App = () => {
  const $ = _c(2);
  let t0;
  if ($[0] === Symbol.for('react.memo_cache_sentinel')) {
    t0 = [1, 2, 3];
    $[0] = t0;
  } else {
    t0 = $[0];
  }
  let t1;
  if ($[1] === Symbol.for('react.memo_cache_sentinel')) {
    t1 = (
      <HeavyComponent arrayProp={t0} callbackProp={_temp}>
        <ChildComponent />
      </HeavyComponent>
    );
    $[1] = t1;
  } else {
    t1 = $[1];
  }
  return t1;
};

function _temp() {
  console.log('Callback happened');
}
```

Several things to notice about this transformation:

1. **The inline callback was hoisted** to a `_temp` function outside the component entirely. This means it never gets recreated on re-render.
2. **`ChildComponent` and `HeavyComponent` are grouped together** into a single cached variable `t1`, rather than being memoized separately. The Compiler determined they can be treated as a unit.
3. **A cache array `$`** (initialized via `_c(2)`) stores memoized values using sentinel values to detect first render vs. subsequent renders.

You can experiment with transformations yourself using the online [React Compiler Playground](https://playground.react.dev/).

## Performance Impact: Measured Results

The Compiler's impact can be predicted from understanding what it does: it adds memoization (improving [interaction performance](interaction-performance.md)) but generates more code (potentially affecting [initial load](web-performance-metrics.md) and [bundle size](bundle-size-optimization.md)).

### Bundle Size Impact

Measurements on a study project showed bundle sizes increasing, but not dramatically:

| Bundle | Before | After | Change |
|--------|--------|-------|--------|
| index.js | 4.23 kB | 5.97 kB | +41% |
| message-editor-fixed.js | 4.96 kB | 11.68 kB | +135% |
| inbox.js | 9.58 kB | 12.90 kB | +35% |
| settings.js | 11.93 kB | 16.55 kB | +39% |
| dashboard.js | 12.55 kB | 17.45 kB | +39% |
| index (main).js | 13.78 kB | 19.97 kB | +45% |
| sidebar-spa.js | 18.35 kB | 20.94 kB | +14% |
| vendor.js | 205.12 kB | 211.00 kB | +3% |
| editor.js | 295.79 kB | 295.79 kB | 0% |

The percentage increases look significant for small bundles, but in absolute terms these are modest additions. Note that vendor bundles and large third-party code (editor, radix) barely change or stay the same -- the Compiler only transforms **your** code, not library code. These are also uncompressed sizes; after gzip/brotli compression, the differences shrink further.

### Interaction Performance (INP) Impact

| Metric | Baseline | With Compiler | Change |
|--------|----------|---------------|--------|
| "Home" page LCP | 1.3 s | 1.4 s | +0.1 s (worse) |
| "Statistics" table load | 2.3 s | 2.5 s | +0.2 s (worse) |
| **Search field INP** | **650 ms** | **50 ms** | **-92% (13x better)** |
| Home INP from Login | 353 ms | 355 ms | unchanged |
| Home INP from Settings | 263 ms | 260 ms | unchanged |

*(Measurements taken with 20x CPU slowdown and Fast 4G network throttling)*

The results confirm both predictions:

- **Initial load worsened slightly** due to increased JavaScript processing. However, this was only measurable at extreme (20x) CPU throttling. At 6x slowdown, the difference disappeared entirely.
- **The search interaction improved by more than 13x**, from 650 ms to 50 ms INP. The Compiler completely solved the [re-render](react-re-renders.md) problem for this use case -- matching what manual memoization would have achieved.
- **Page transitions were unaffected.** Navigation from Login to Home and from Settings to Home showed no improvement, which reveals important limitations (discussed below).

## Limitations

The Compiler cannot catch everything. In real-world testing, it tends to catch **40 to 80% of re-renders** on a page, with varying degrees of improvement.

### Limitation 1: Code That Uses State Internally

When navigation items re-rendered despite the Compiler being enabled, investigation revealed the cause: each item used a `usePath()` hook that set state on every URL change. Since [re-renders are triggered by state changes](react-re-renders.md), and this hook lives inside each item component, every navigation causes every item to re-render -- not because memoization failed, but because the state change is happening **inside** the component.

The Compiler correctly memoized these components (confirmed by the "Memo" label in React DevTools), but memoization does not prevent re-renders caused by internal state changes. This is a fundamental aspect of how React works, not a Compiler bug. The fix requires restructuring code to minimize where state lives -- a design concern outside the Compiler's scope.

**Debugging approach when something is not optimized:**

1. Check if the Compiler applied memoization (look for "Memo" labels in React DevTools Components tab).
2. Verify re-renders actually happen (add `console.log` inside `useEffect` with no dependencies).
3. Investigate **why** by commenting out hooks and code blocks to isolate the trigger.

### Limitation 2: External Libraries Are Not Compiled

The Compiler transforms **your code only**. Everything pulled in as a dependency is at the mercy of its maintainers. If a library does not run its code through the Compiler before distribution, its components will not be automatically memoized.

This is especially important for UI component libraries like MUI, Ant Design, Radix, or any other component framework. In the study project (built on Radix UI), only the wrapper components written in app code received the "Memo" label. All internal Radix primitives used under the hood remained unoptimized.

In the DevTools Components tab, you would see a picture where only `TabsList` and `TabButton` (app-level wrappers) have the "Memo" label, while all the Radix internals between them do not.

### Limitation 3: Complex Code That Defeats Static Analysis

Sometimes the Compiler simply fails to optimize correctly. JavaScript is notoriously flexible and full of patterns that are difficult for static analysis to handle.

Consider this pattern with a hover-controlled list:

```jsx
export const MessageListFixed = () => {
  const [hoveredMessage, setHoveredMessage] = useState(null);

  return messages.map((message) => (
    <div
      onMouseEnter={() => setHoveredMessage(message.id)}
      onMouseLeave={() => setHoveredMessage(null)}
    >
      <Checkbox />
      <Icons.Star />
      {hoveredMessage === message.id ? <div>content</div> : null}
    </div>
  ));
};
```

Despite `Checkbox` and `Icons.Star` having no dependency on `hoveredMessage`, they re-render on every hover. The Compiler grouped the entire `map` callback -- including the independent components and the conditional div -- into a single cached variable. Since that variable depends on `hoveredMessage`, the entire group invalidates on every hover:

```jsx
// Compiler output (simplified)
let t0;
if ($[0] !== hoveredMessage) {
  t0 = messages.map((message) => (
    <div onMouseEnter={...} onMouseLeave={...}>
      <Checkbox />
      <Icon />
      {hoveredMessage === message.id ? <div>content</div> : null}
    </div>
  ));
  $[0] = hoveredMessage;
  $[1] = t0;
} else {
  t0 = $[1];
}
return t0;
```

The Compiler optimized for grouping efficiency but lost the ability to memoize `Checkbox` and `Icons.Star` independently from the conditional content. The simpler and cleaner the components, the easier it is for the Compiler to correctly memoize them. This particular issue stems from the component being too large and doing too many things at once -- a case where [component composition patterns](component-composition-patterns.md) would help the Compiler succeed.

Note that the React team constantly works on improving the Compiler, and specific issues like these may be fixed in newer versions. However, since it is parsing arbitrary JavaScript, there will always be edge cases.

## Is It Worth It?

The answer depends on context:

**Arguments for enabling the Compiler:**
- It is a performance improvement "for free" when it works -- no manual memoization effort required.
- It eliminates the need for tedious and error-prone manual `useMemo`/`useCallback`/`React.memo` patterns.
- Setup is straightforward on compatible build systems.
- Impact on initial load is minimal, even negligible under normal CPU conditions.
- For apps with inconsistent or broken memoization, it provides immediate and consistent optimization.

**Arguments for caution:**
- It introduces more "magic" into how React code behaves.
- You can no longer tell whether a component will re-render just by looking at the source code -- it depends on whether the Compiler succeeded in its transformation.
- Debugging becomes more challenging when something goes wrong.
- It is still in Release Candidate / experimental status.
- It catches only 40-80% of re-renders in practice, so manual optimization is still needed for the rest.

**When the Compiler helps most:**
- Applications with significant, unfixed memoization issues.
- Simple, clean component code that the Compiler can analyze effectively.
- Interactions that suffer from cascading [re-renders](react-re-renders.md) (like the search field example with 13x improvement).

**When it helps least:**
- Apps already well-optimized through [composition patterns](component-composition-patterns.md) and proper state placement.
- Complex or messy components that defeat static analysis.
- Heavy reliance on external UI libraries that are not pre-compiled.

## Relationship to Manual Memoization

The Compiler does not eliminate the need to understand [memoization](memoization.md) and [re-renders](react-re-renders.md). It automates the mechanical part -- wrapping values in `useMemo` and callbacks in `useCallback` -- but understanding **why** re-renders happen and how to structure components to minimize them remains essential. The Compiler cannot fix architectural problems like having all state at the top of the tree, using hooks that trigger state changes inside every component, or relying on uncompiled external libraries.

Think of the Compiler as a safety net for the memoization patterns covered in [Memoization in React](memoization.md), not as a replacement for the structural optimization techniques described in [Component Composition Patterns](component-composition-patterns.md) and [React Re-renders](react-re-renders.md).

## Key Takeaways

- The React Compiler is a **Babel plugin**, not part of React itself. It requires separate installation and a compatible build system.
- It supports **React 17+**, so teams on older versions can still benefit.
- It automatically memoizes components at build time, producing optimized code that uses a cache array and sentinel values rather than literal `useMemo`/`useCallback` wrappers.
- **Interaction performance can improve dramatically** -- up to 13x in measured cases -- by eliminating unnecessary re-renders.
- **Initial load performance may degrade slightly** due to increased bundle size and JavaScript processing, but the effect is negligible under normal conditions.
- **Bundle size increases** are modest in absolute terms, especially after compression, and only affect your code (not third-party libraries).
- The Compiler **cannot catch everything**: internal state changes, external libraries, and complex code patterns all remain blind spots.
- In practice, it catches **40-80% of re-renders** on a page.
- **Simpler, cleaner components** are easier for the Compiler to optimize. Large, monolithic components that do too many things tend to defeat its analysis.
- Use the **ESLint plugin** to identify code that violates React rules and would prevent the Compiler from optimizing correctly.
- Enabling the Compiler trades predictability for automation -- you gain automatic optimization but lose the ability to determine re-render behavior from source code alone.

## See Also

- [Memoization in React](memoization.md) -- the manual patterns the Compiler automates
- [React Re-renders](react-re-renders.md) -- understanding when and why components re-render
- [Reconciliation and Diffing](reconciliation-and-diffing.md) -- the process that memoization short-circuits
- [Component Composition Patterns](component-composition-patterns.md) -- structural alternatives that complement the Compiler
- [Web Performance Metrics](web-performance-metrics.md) -- LCP, INP, and other metrics referenced in measurements
- [Interaction Performance](interaction-performance.md) -- the performance dimension most improved by the Compiler
- [Bundle Size Optimization](bundle-size-optimization.md) -- context for understanding the Compiler's size trade-off
