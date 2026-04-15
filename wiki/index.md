# Knowledge Base Index

## react

Advanced React patterns, performance optimization, and web performance fundamentals for React applications.

### Core Rendering Model

How React decides what to render, how re-renders propagate, and how to structure components to control that flow.

| Article | Summary | Updated |
|---------|---------|---------|
| [React Re-renders](react/react-re-renders.md) | How re-renders work, triggers, propagation, the big myth, and prevention patterns | 2026-04-13 |
| [Reconciliation and Diffing](react/reconciliation-and-diffing.md) | Virtual DOM diffing algorithm, position-based comparison, key attribute, state reset and force-reuse techniques | 2026-04-13 |
| [Component Composition Patterns](react/component-composition-patterns.md) | Elements as props, children, render props, HOCs, compound components, and flexible layout patterns | 2026-04-13 |

### Hooks & Patterns

Specific React APIs and common coding patterns for everyday development.

| Article | Summary | Updated |
|---------|---------|---------|
| [Memoization in React](react/memoization.md) | useMemo, useCallback, React.memo — when to use, pitfalls, children problem, progressive optimization case study, and anti-patterns | 2026-04-13 |
| [Refs and Imperative API](react/refs-and-imperative-api.md) | useRef, forwardRef, useImperativeHandle, callback refs, and DOM measurement patterns | 2026-04-13 |
| [Closures in React](react/closures-in-react.md) | Stale closure problem in hooks, event handlers, timers, and Refs; escape patterns and mental models | 2026-04-13 |
| [Debouncing and Throttling](react/debouncing-throttling.md) | Correct debounce/throttle with Refs, the closure escape pattern, and useDebounce hook extraction | 2026-04-13 |
| [useLayoutEffect](react/use-layout-effect.md) | Flicker prevention, browser rendering pipeline (tasks vs painting), useEffect vs useLayoutEffect timing, and SSR considerations | 2026-04-15 |
| [React Portals](react/react-portals.md) | Portals for escaping CSS stacking contexts, event bubbling vs DOM events, form submission caveat | 2026-04-13 |
| [Error Handling in React](react/error-handling.md) | ErrorBoundary, try/catch limitations, async errors, strategic placement, and recovery patterns | 2026-04-13 |
| [React Compiler](react/react-compiler.md) | Automated memoization via Babel plugin, code transformation examples, measured performance impact, limitations, and compatibility | 2026-04-13 |

### State & Data

Managing application state across different categories and fetching data effectively.

| Article | Summary | Updated |
|---------|---------|---------|
| [React State Management](react/state-management.md) | State categories (remote/URL/local/shared), TanStack Query + nuqs + Zustand stack, Context limitations, and library evaluation framework | 2026-04-14 |
| [React Context and Performance](react/react-context-performance.md) | Context for state distribution, re-render implications, splitting strategies, and higher-order component optimization | 2026-04-14 |
| [Data Fetching Patterns](react/data-fetching-patterns.md) | Client-side fetching, request waterfalls, browser limits, Suspense, SSR vs RSC strategies | 2026-04-14 |
| [Race Conditions in React](react/race-conditions.md) | Stale promise problem, cleanup flags, Ref comparison, AbortController, and forced re-mounting | 2026-04-15 |

### Performance

Measuring, diagnosing, and optimizing web performance in React applications.

| Article | Summary | Updated |
|---------|---------|---------|
| [Web Performance Metrics](react/web-performance-metrics.md) | Core Web Vitals (FCP, LCP, INP), CrUX, RUM, flame graphs, Lighthouse, and measurement methodology | 2026-04-13 |
| [Interaction Performance](react/interaction-performance.md) | INP measurement, SPA transitions, Long Tasks, yielding (and when NOT to), dev vs prod mode for debugging, React DevTools Profiler, and re-render-driven optimization | 2026-04-13 |
| [Bundle Size Optimization](react/bundle-size-optimization.md) | Code splitting, tree shaking (ESM vs CJS), bundle analysis, HTTP/1 vs HTTP/2, transitive deps, duplicate libraries, TTI+SSR gap, and compression trade-offs | 2026-04-13 |
| [Lazy Loading and Suspense](react/lazy-loading-and-suspense.md) | React.lazy, Suspense boundaries, critical path extraction, preloading strategies, framework evaluation checklist, and vendor chunk trade-offs | 2026-04-13 |

### Architecture

Application-level rendering strategies and server-side patterns.

| Article | Summary | Updated |
|---------|---------|---------|
| [Rendering Strategies](react/rendering-strategies.md) | CSR, SSR, SSG, hydration mechanics, streaming, and framework trade-offs | 2026-04-13 |
| [React Server Components](react/react-server-components.md) | RSC architecture, serialized element trees, streaming with Suspense, async components, "use client" boundary rules, and performance comparison vs CSR/SSR | 2026-04-13 |

### Interview Questions

Curated question-and-answer sets for React interview preparation, organized by topic area.

| Article | Summary | Updated |
|---------|---------|---------|
| [React Fundamentals](react/react-fundamentals-interview.md) | What is React and its benefits, React Node vs Element vs Component, JSX transformation and key characteristics | 2026-04-14 |
| [React Hooks](react/react-hooks-interview.md) | Benefits of hooks, rules of hooks, useState callback, useEffect dependency array, useEffect vs useLayoutEffect, useRef, useCallback, useMemo, useReducer, useId | 2026-04-14 |
| [React Components & State](react/react-components-interview.md) | State vs props, key prop purpose, index keys pitfall, controlled vs uncontrolled, fragments, resetting state, immutability, error boundaries | 2026-04-14 |
| [React Architecture & Performance](react/react-architecture-interview.md) | Re-rendering, Virtual DOM, reconciliation, Context pitfalls/optimization, HOCs, composition, code splitting, Suspense, hydration, SSR, SSG, portals, testing, state manager decision guide | 2026-04-14 |
