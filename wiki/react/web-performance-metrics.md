# Web Performance Metrics

> Sources: Nadia Makarevich, 2025
> Raw: [Web Performance Fundamentals](../../raw/react/web-performance-fundamentals-nadia-makarevich.md)

## Overview

Web performance is measurable through specific metrics that correlate with user experience and business outcomes. Research consistently shows that even 100ms improvements in site speed lead to measurable business gains. The measurement pipeline starts with free tools like CrUX, moves to Real User Monitoring for granular data, and uses Chrome DevTools for local profiling. Understanding how to read flame graphs, simulate network conditions, and correlate metrics with the rendering timeline is essential for diagnosing and fixing performance issues.

## Why Performance Matters

Performance directly impacts business metrics. A Deloitte study analyzing four weeks of mobile data across European and US brands found that even a 100ms improvement in site speed results in improved funnel progression, page views, bounce rate, and conversion rate.

Notable case studies from Google's web.dev:

- **Vodafone**: 31% LCP improvement led to 8% increase in total sales, 15% improvement in lead-to-visit rate, 11% improvement in cart-to-visit rate
- **QuintoAndar** (Brazil's largest housing platform): 80% INP reduction led to 36% increase in conversions
- **Disney+ Hotstar**: 61% INP reduction led to 100% increase in weekly card views
- **Shopify**: Confirmed in 2024 that performance still matters and continues investing heavily

Google Search uses Core Web Vitals data to influence search rankings, making performance directly tied to discoverability.

## Core Web Vitals

Google defines three Core Web Vitals that represent different aspects of user experience:

### Largest Contentful Paint (LCP)

Time until the largest visible content element (image, text block, video) finishes rendering in the viewport. Represents when the main content is visually complete. According to Google, LCP should be below **2.5 seconds**. This is the Core Web Vital responsible for **loading performance**.

LCP depends on the entire critical rendering path: TTFB, render-blocking resources, JavaScript evaluation, and layout calculation.

### Interaction to Next Paint (INP)

Measures responsiveness: the latency from user interaction (click, tap, keyboard) to the next visual update. Replaced First Input Delay (FID) as a Core Web Vital in 2023. Google considers INP "good" if below **200ms**, "needs improvement" between 200-500ms, and "poor" above 500ms.

An interaction includes three phases:
- **Input delay**: time waiting for the event handler to start (blocked by other tasks)
- **Processing time**: event handler execution
- **Presentation delay**: rendering, layout, and painting after the handler completes

In SPAs, navigation between pages is often the largest INP number, which means other interactions can be overlooked if you hyper-focus on the overall score.

Important detail: INP is tracked per page. SPA transitions keep the score between pages, but non-SPA navigations (full page reloads) reset the score.

### First Contentful Paint (FCP)

Time from navigation start to when the browser renders the first piece of DOM content (text, image, SVG, non-white canvas). According to Google, a good FCP is below **1.8 seconds**. FCP measures perceived initial load — the user's first impression of speed.

FCP is imperfect: if the website starts with a spinner or loading screen, FCP captures that, not the actual content. This is why LCP is often more meaningful.

## Other Important Metrics

### Time to First Byte (TTFB)

Time from when the request is sent to when the first byte of the response arrives from the server. Not a Core Web Vital, but foundational — all other metrics are bottlenecked by TTFB. A slow server (500ms delay before returning HTML) can dominate the performance picture, making frontend optimizations negligible.

### Time to Interactive (TTI)

The time when the page becomes fully interactive. Particularly important for SSR, where a page can be visible (FCP) but non-functional until JavaScript loads and hydration completes. TTI was removed from Lighthouse and Core Web Vitals but remains useful for analyzing SSR performance.

## The Critical Rendering Path

When a browser navigates to a page, the rendering follows a specific pipeline:

1. **Request HTML** from the server (TTFB)
2. **Parse HTML** — extract DOM structure and discover linked resources
3. **Fetch render-blocking resources** — CSS in `<link>` tags, synchronous JS in `<head>` (not `async` or `defer`; `type="module"` scripts are deferred automatically)
4. **Process blocking resources** — parse CSS, evaluate blocking JS
5. **First Paint / FCP** — the browser can now paint something meaningful
6. **Fetch and evaluate remaining JS** — non-blocking scripts, React bundles
7. **LCP** — the largest content element finishes rendering

Render-blocking resources are the gatekeepers: nothing paints until they are loaded. CSS is almost always blocking. JavaScript in `<head>` without `async` or `defer` is blocking. Understanding which resources are blocking is the first step to improving FCP and LCP.

## Measurement Tools

### CrUX (Chrome User Experience Report)

Real-world performance data collected from Chrome users browsing public websites. Available as weekly visualizations, dashboards, BigQuery datasets, and a developer API. CrUX is invaluable for:

- Free overview of Core Web Vitals and historical trends
- Comparing your site against competitors (data is public)
- Seeing the impact of performance improvement initiatives

**Limitations:**
- Measures only Core Web Vitals (Google's chosen set)
- Chrome-only, eligible users only — some user segments are excluded
- Public websites only — useless behind authentication

### Real User Monitoring (RUM)

Application-level monitoring that collects performance data from actual users. More granular than CrUX — custom metrics, page-specific data, user segmentation. This is the crucial step: measuring real customers is the only way to identify whether you actually have performance problems worth solving.

Two parts needed:
1. **Extraction**: Something that captures performance data from the live site. For Core Web Vitals alignment, use Google's `web-vitals` library. For custom metrics, combine your analytics tool's client with the JavaScript Performance API.
2. **Dashboarding**: A system to receive, process, and visualize the data — Google Analytics, Sentry, Datadog, New Relic, etc.

### Lighthouse

Automated auditing tool integrated into Chrome DevTools (also available as CLI, web interface, and Node module). Provides performance scores and actionable recommendations.

- **Navigation mode**: analyzes initial page load (FCP, LCP, etc.)
- **Timespan mode**: analyzes interactions during a recording period (INP)

Lighthouse is great as an entry point and for tracking changes over time, but it runs in a simulated environment. Scores may not match real-world conditions. Use it for relative comparisons, not absolute truth.

### Chrome DevTools Performance Panel

The most powerful local profiling tool. Provides:

- **Timeline overview**: bird's-eye view of activity over time
- **Network section**: all downloaded resources with exact timing, blocking indicators (red corners)
- **Frames section**: screenshots of what was rendered at each point in time
- **Main section**: detailed flame graph of main thread activity
- **Interactions section**: INP recordings with exact durations and origins
- **Metrics**: FCP, LCP markers on the timeline with Summary tab details

Key settings for profiling:
- **Screenshots checkbox**: correlate visual changes with timeline events
- **Disable cache**: emulate first-time visitors
- **CPU throttle**: essential for interaction profiling (use 6x or 20x slowdown — developer laptops mask real-world issues)
- **Network throttle**: simulate different connection qualities

## Flame Graphs

Flame graphs visualize the call stack over time. The key concepts:

- **X-axis**: time progression (left to right)
- **Y-axis**: call stack depth (deeper = more nested function calls)
- **Width of a bar**: duration of that function
- **Colors**: different categories — yellow (JavaScript), purple (layout/style), green (painting), blue (HTML parsing)

### Total Time vs Self Time

- **Total Time**: time for a function including all its children
- **Self Time**: Total Time minus children's execution time — the time spent in the function itself

This distinction is critical for optimization decisions. If a function has 30ms Total Time but only 5ms Self Time, the problem is in the children, not the function itself.

### Reading the DevTools Flame Graph

When analyzing performance graphs:

1. Identify which tasks are the bottleneck (longest bars)
2. Click on bars to see "Total time", "Self time", and "Function" link in the Summary tab
3. Use the Function link to navigate to the exact source code
4. Watch for Chrome plugin interference — always profile in Incognito/Guest mode to eliminate third-party extensions that add JavaScript to every page

### Practical Skills

- Identify long tasks (>50ms, shown with red corner)
- Find JavaScript evaluation time vs execution time
- Spot network waterfalls (chained downloads)
- Correlate rendering events with metric markers (FCP, LCP)
- Distinguish between render-blocking resources and non-blocking ones

## Network Conditions and Their Impact

Performance varies dramatically with network conditions. Key variables:

- **Bandwidth**: download/upload speed (affects how fast files transfer)
- **Latency**: round-trip delay before data even starts moving (affects every request)

High latency trumps everything. A 1Gbps connection with 300ms latency can be worse than a 10Mbps connection with 40ms latency. This is because every resource (HTML, CSS, JS) must pay the latency penalty.

### Testing Scenarios

| Scenario | Bandwidth | Latency | Use Case |
|----------|-----------|---------|----------|
| No throttle | Full | ~0 | Desktop baseline |
| Fast 4G | 50 Mbps | 40ms | Typical urban mobile |
| Slow 3G | 10 Mbps | 300ms | Rural/developing regions |
| High latency | 1 Gbps | 300ms | Fast device, remote server |

### The Importance of CDN

CDN is "step zero" in frontend performance — before code splitting, Server Components, or any fancy optimization. A CDN reduces latency by distributing static resources to servers geographically close to users. In the high-latency scenario above (servers in Norway, users in Australia), adding a CDN with an Australian edge server dropped LCP from 960ms back to 640ms.

## Repeat Visit Performance and Caching

Repeat visitors benefit from browser caching, but caching behavior depends on the `Cache-Control` header.

### Cache-Control Basics

- `max-age=0, must-revalidate`: Browser always checks with server (304 Not Modified responses still incur latency)
- `max-age=31536000`: Browser uses cached version for up to one year without checking the server
- `(memory cache)` or `(disk cache)`: Resource served locally, zero network cost

### Cache-Busting with Content Hashes

Modern bundlers (Vite, Webpack, Rollup) generate file names with content hashes: `index-Cx2U5bbX.js`. When the file content changes, the hash changes, and the browser fetches a fresh copy. This means you can safely set `max-age=31536000` (one year) for all generated assets — the cache is automatically "busted" when code changes.

Best practice: set maximum `max-age` for hashed static assets (JS, CSS, images). This gives free performance boost for repeat visitors on every deployment.

## See Also

- [Rendering Strategies](rendering-strategies.md)
- [Bundle Size Optimization](bundle-size-optimization.md)
- [Interaction Performance](interaction-performance.md)
- [Lazy Loading and Suspense](lazy-loading-and-suspense.md)
