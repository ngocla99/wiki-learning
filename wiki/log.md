# Wiki Log

## [2026-04-14] lint | Full wiki audit (23 articles)

**Deterministic checks — all passed, 2 auto-fixes applied:**

- Fixed: `lazy-loading-and-suspense.md` — removed `./` prefix from See Also links and 1 inline body link; normalized to bare filenames
- Fixed: `bundle-size-optimization.md` — removed `./` prefix from See Also links; normalized to bare filenames; also removed inline `-- description` text to match style of all other articles
- Index consistency: all 23 articles in index.md have matching files ✓
- Internal links: all See Also and inline body links resolve to existing files ✓
- Raw references: all Source/Raw fields point to existing raw files ✓

**Heuristic checks — issues reported, no auto-fix:**

- Orphan inbound links: the 4 new interview Q&A articles (react-fundamentals-interview.md, react-hooks-interview.md, react-components-interview.md, react-architecture-interview.md) are not linked from any existing deep-dive article; discoverable only via index.md
- Missing sibling links: `react-fundamentals-interview.md` See Also does not reference the other 3 interview articles in the series (the other 3 cross-link to each other)

## [2026-04-14] ingest | React Interview Questions (extended set)

- Source: [GreatFrontEnd React Interview Questions](https://www.greatfrontend.com/questions/quiz/react-interview-questions) (GreatFrontEnd, 34 questions)
- New: React Hooks — Interview Questions
- New: React Components & State — Interview Questions
- New: React Architecture & Performance — Interview Questions
- Updated: React Fundamentals — Interview Questions (added cross-references)

## [2026-04-14] ingest | React Fundamentals — Interview Questions

- Source: [GreatFrontEnd React Interview Questions](https://www.greatfrontend.com/questions/quiz/react-interview-questions) (GreatFrontEnd)
- New: React Fundamentals — Interview Questions

## [2026-04-14] ingest | React State Management

- Source: [React State Management in 2025](https://www.developerway.com/posts/react-state-management-2025) (Nadia Makarevich, 2025-09-25)
- New: React State Management
- Updated: React Context and Performance (added cross-reference)
- Updated: Data Fetching Patterns (added cross-reference)

## [2026-04-13] ingest | Raw source update re-ingest

- Source: Web Performance Fundamentals (Nadia Makarevich, 2025) -- updated raw extraction (~35KB new content)
- Updated: Bundle Size Optimization (ESM vs CJS tree shaking, transitive deps, HTTP/1 vs HTTP/2, TTI+SSR gap)
- Updated: Lazy Loading and Suspense (framework evaluation checklist, TanStack Start, vendor chunk trade-offs)
- Updated: Interaction Performance (dev vs prod mode clarification, yield caveat)
- Updated: Memoization in React (progressive search field optimization case study)

## [2026-04-13] rewrite | Full React wiki expansion

- Rewrote all 18 React articles from ~60-120 lines each to 200-640 lines each
- Source: Advanced React (Nadia Makarevich, 2023) -- chapters 1-16
- Source: Web Performance Fundamentals (Nadia Makarevich, 2025) -- chapters 1-14
- Added comprehensive code examples, performance measurements, and key takeaways to every article
- Added cross-references (See Also) between all related articles
- Updated index.md with expanded summaries

## [2026-04-09] ingest | React Re-renders

- Source: Advanced React (Nadia Makarevich, 2023)
- Source: Web Performance Fundamentals (Nadia Makarevich, 2025)
- New: Component Composition Patterns
- New: Memoization in React
- New: Reconciliation and Diffing
- New: React Context and Performance
- New: Refs and Imperative API
- New: Closures in React
- New: Debouncing, Throttling, and useLayoutEffect
- New: React Portals
- New: Data Fetching Patterns
- New: Error Handling in React
- New: Web Performance Metrics
- New: Rendering Strategies
- New: Bundle Size Optimization
- New: Lazy Loading and Suspense
- New: React Server Components
- New: Interaction Performance
- New: React Compiler
