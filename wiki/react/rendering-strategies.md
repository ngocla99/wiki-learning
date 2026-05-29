# Rendering Strategies

> Sources: Nadia Makarevich, 2025; Vu Nguyen (upskills.dev), 2026
> Raw: [Web Performance Fundamentals](../../raw/react/web-performance-fundamentals-nadia-makarevich.md)
> Raw: [React Rendering Strategies — upskills.dev](../../raw/react/2026-03-02-react-rendering-strategies-upskills.md)

## Overview

Modern React applications can be rendered using several strategies: Client-Side Rendering (CSR), Server-Side Rendering (SSR), Static Site Generation (SSG), and hybrid approaches. Each has distinct trade-offs for performance, SEO, interactivity, and operational complexity. The choice depends on the application's priorities — and understanding the mechanics of each strategy is the only way to make an informed decision rather than following hype.

## Why These Patterns Exist

Each rendering pattern emerged because the previous approach hit a real production constraint:

- **Pre-2010 server-rendered (.NET MVC, Rails, PHP)**: every interaction triggered a full round-trip and full page reload. HTML generation was tightly coupled to business logic. Interactivity was limited.
- **Late 2000s jQuery era**: kept the server in charge of HTML, sprinkled in `$.ajax()` and DOM manipulation. The pattern broke down when teams tried to "build applications with a tool designed for enhancements."
- **2010–2013 first SPA wave (Backbone, Knockout, AngularJS, Ember)**: introduced declarative bindings and client-side routing. Two-way binding made small demos easy and large apps "nightmarish to debug at scale."
- **2013 React**: state changes → React re-renders → only the changed parts of the DOM update. Predictable, composable, won the market.
- **Post-2016 return to the server**: SPA trade-offs surfaced — SEO challenges, slow initial loads, loading spinners, bundle bloat, waterfall fetches. The community brought server rendering back, this time as a layer on top of React rather than a replacement.

The takeaway: there is no "best" pattern in the abstract. Each one is a response to specific problems the previous generation could not solve, and choosing well means knowing which of those problems you actually have.

## Client-Side Rendering (CSR)

CSR is the default rendering pattern for React apps built with tools like Vite or Create React App. The server sends a minimal HTML shell, and JavaScript takes over to generate the full UI.

### How It Works

1. Server sends bare HTML with an empty `<div id="root">` and `<script>` tags
2. Browser downloads and evaluates the JavaScript bundle
3. React mounts, creates DOM elements, and appends them to the root div
4. User sees content only after JavaScript completes

```html
<html>
  <head>
    <script type="module" src="/assets/index-Cx2U5bbX.js"></script>
    <link rel="stylesheet" href="/assets/index-BjPt9w-2.css" />
  </head>
  <body>
    <div id="root"></div>
  </body>
</html>
```

The entry point in React is straightforward:

```jsx
createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>,
);
```

### Performance Characteristics

In the flame graph, CSR shows a clear pattern: blue "Parse HTML" bar, then long yellow JavaScript bars, and only after JavaScript completes do FCP and LCP trigger at the same time. Content is invisible until JavaScript finishes.

Compare with a traditional server-rendered site (like MDN docs): blue "Parse HTML" is followed by purple "Layout" and green "Painting" immediately, with FCP/LCP triggered before any significant JavaScript runs.

**The cost of CSR is clear**: the initial load and LCP will always suffer because the browser must wait for JavaScript to execute before showing anything. Without JavaScript, users get a blank page.

### Why CSR Is Still Valuable

1. **Simple and cheap**: Static files deployable anywhere (CDN, GitHub Pages) at zero hosting cost. No server to manage, no scaling concerns, no memory leaks.
2. **Initial load is not always the priority**: For SaaS applications (project management, dashboards, email clients), users open the app to interact with it, not to admire the initial render. They may tolerate a slightly slower load for excellent interaction performance afterward.

## Single-Page Applications (SPAs)

SPAs are a natural extension of CSR where client-side routing handles navigation without full page reloads.

### How SPA Navigation Works

Under the hood, SPA navigation uses the History API:

```jsx
// Inside a Link component
<a onClick={(e) => {
  e.preventDefault();
  navigate(href);
}}>
  {children}
</a>

// navigate() does this:
window.history.pushState({}, '', newPath);
dispatchEvent(new PopStateEvent('popstate', { state: {} }));
```

A listener picks up the event and updates React state:

```jsx
window.addEventListener('popstate', () => {
  setPath(window.location.pathname);
});
```

Then a router renders the appropriate page based on the path:

```jsx
switch (path) {
  case "/login": return <LoginPage />;
  default: return <DashboardPage />;
}
```

### SPA vs Traditional Navigation in the Performance Panel

**SPA transition**: Empty Network section (no requests), a spike of JavaScript activity in Main, purple layout/green paint blocks, then the new page appears. No FCP/LCP metrics. Entire transition takes ~60ms.

**Traditional navigation (full page reload)**: FCP/LCP metrics return, Network shows new HTML/CSS/JS requests, full Parse HTML cycle. The same transition takes 10x longer (~1 second on slow 3G).

These fast, snappy transitions are what make SPAs so attractive. They work at any granularity — you can make individual tabs feel like navigation, not just full pages.

## Server-Side Rendering (SSR)

The server runs React, generates full HTML, and sends it to the browser. The browser shows content immediately, then JavaScript loads and "hydrates" the page with interactivity.

### How It Works

React provides `renderToString` to generate HTML on the server:

```jsx
const App = () => <div>React app</div>;
// On the server:
const html = renderToString(<App />);
// Output: <div>React app</div>
```

The server injects this HTML into the root div and sends it to the browser:

```jsx
app.get('/*', async (c) => {
  const html = fs.readFileSync('dist/index.html').toString();
  const reactHTML = renderToString(<App />);
  const finalHTML = html.replace('<!--ssr-->', reactHTML);
  return c.html(finalHTML);
});
```

### Impact on Performance Metrics

- **FCP and LCP improve significantly** — content is in the HTML before JavaScript loads
- **TTFB may worsen** — the server must run React and possibly fetch data before responding
- **Time to Interactive is delayed** — the visible page is non-functional until hydration completes

In the flame graph, SSR shows: blue HTML download, then almost immediately a purple "Layout" block and FCP trigger (content visible). JavaScript still downloads in the background. Only after JavaScript evaluates does the page become interactive.

### SSR Can Make LCP Worse

SSR is not a universal win. With a slow network, high latency, and cached CSS/JS files (repeat visitors), the large SSR HTML takes longer to download than the small empty-div CSR HTML. In one test scenario:

- **CSR LCP**: 2.13 seconds (small HTML, cached JS executes fast)
- **SSR LCP**: 2.62 seconds (large HTML downloads slowly over constrained bandwidth)

The combination of slow network + huge latency + fast laptop occurs for business travelers, remote workers, and users in areas with high-latency connections. The lesson: know your customers and measure everything.

### Hydration

Hydration is how React takes control of server-rendered DOM. Instead of clearing the root div and recreating everything, React traverses the existing DOM, attaches event handlers, and initializes state:

```jsx
hydrateRoot(
  document.getElementById('root')!,
  <StrictMode>
    <App />
  </StrictMode>,
);
```

Without hydration, React destroys the SSR'd content and recreates it (wasting the SSR work). With hydration, React reuses the existing DOM nodes — faster and no visual flash.

**Key rules:**
- `useEffect` and `useLayoutEffect` do not run during SSR — they execute only on the client after hydration
- Server and client HTML must match exactly. Conditional rendering based on `typeof window === "undefined"` breaks hydration and causes React to fall back to CSR behavior
- The correct pattern for client-only content:

```jsx
const Component = () => {
  const [isMounted, setIsMounted] = useState(false);
  useEffect(() => { setIsMounted(true); }, []);
  if (!isMounted) return null;
  return <ClientOnlyContent />;
};
```

### SSR and Frontend Challenges

**Browser APIs**: `window`, `document`, etc. are undefined on the server. Direct access causes "window is not defined" crashes on the server. Guard with `typeof window !== 'undefined'`.

**Third-party libraries**: Not all libraries support SSR. Some will crash the server, some need dynamic imports, some require replacement with SSR-friendly alternatives. A non-SSR-compatible foundational library (state management, CSS-in-JS) can be especially painful.

### Should You Implement SSR Yourself?

No. The study project implementation hides half the complexity and doesn't support features like:
- Dev server with hot reload for SSR
- `renderToPipeableStream` (streaming and Suspense support)
- Proper chunk injection for lazy-loaded components
- Data hydration

Use an existing framework: Next.js, React Router (framework mode), or TanStack Start. Building your own SSR is essentially building your own framework.

## The Cost of Having a Server

Moving from a pure static SPA to any server-rendered setup introduces two problems:

### 1. Deployment Complexity and Cost

Options:
- **Serverless functions** (Cloudflare Workers, Netlify/Vercel Functions, AWS Lambda): No server management, pay-per-usage, some run "on Edge" (distributed). Risk: viral traffic can cause surprise bills.
- **Self-managed server** (AWS, Azure, DigitalOcean): Full control, predictable pricing, but you own monitoring, scaling, memory leaks, and geographic distribution.

### 2. Performance Impact

Introducing a server means a mandatory round-trip for every initial load request, even for repeat visitors. If the server is in one region and users are global, latency degrades performance. Edge functions or CDN caching of dynamic responses help mitigate this.

Even with "simple" Next.js deployment to Vercel/Netlify, the framework converts your app into serverless functions behind the scenes — all server-related concerns apply.

## Static Site Generation (SSG)

Pages are rendered at **build time** using `renderToString`, producing static HTML files served directly from a CDN.

```bash
# Build produces login.html, settings.html, etc.
# Each file has <div id="root"> filled with content
npm run build:ssg
```

**Advantages:**
- Fastest possible TTFB (pre-built files from CDN, no server)
- Full SEO support (HTML contains all content)
- No server runtime, no scaling concerns
- Can be hosted anywhere static files are served

**Disadvantages:**
- Cannot personalize per user or per request
- Data is stale until the next build
- Not suitable for highly dynamic content

Frameworks supporting SSG: Next.js (static export), Gatsby, Docusaurus, Astro.

### Incremental Static Regeneration (ISR)

The "data goes stale until the next build" problem has a partial fix: rebuild individual pages on a timer, in the background, while continuing to serve the cached version. This is the **stale-while-revalidate** pattern. When a cached page is older than its revalidation window, the next visitor still gets the stale page instantly; the framework regenerates a fresh version in the background; the visitor *after* that gets the fresh one.

Next.js Pages Router:

```javascript
export async function getStaticProps() {
  return {
    props: { data },
    revalidate: 3600,  // re-generate at most once per hour
  };
}
```

Next.js App Router (per-fetch revalidation):

```javascript
const data = await fetch(url, { next: { revalidate: 3600 } });
```

ISR keeps SSG's performance profile — CDN-served files, zero per-request compute — while letting content stay reasonably fresh. The trade-offs: still no per-user personalization (every visitor gets the same cached page), and revalidation happens lazily on traffic, so a low-traffic page can stay stale longer than its window suggests. ISR is a sweet spot for content that changes occasionally but doesn't need to be live: catalogs, marketing pages, blog posts, documentation sites that pull from a CMS.

## No-JavaScript Environments

Two critical consumers access your HTML without JavaScript:

### Search Engine Crawlers

Google has a two-step process: first it parses pure HTML, then it queues the page for JavaScript rendering in a headless browser. JavaScript-heavy pages get slower and budgeted indexing.

### Social Media Previewers

Link previews in messengers and social platforms extract meta tags from raw HTML only. No JavaScript is loaded.

For websites where discoverability and shareability are critical (blogs, docs, e-commerce, landing pages), the server must return proper HTML with meta tags and content on the first request.

### Server Pre-rendering of Meta Tags

A lightweight alternative to full SSR: modify the HTML string on the server to inject proper `<title>` and Open Graph `<meta>` tags per route:

```jsx
app.get('/*', async (c) => {
  const html = fs.readFileSync('dist/index.html').toString();
  const title = getTitleFromPath(c.req.path);
  return c.html(html.replace('{{ title }}', title));
});
```

This solves social media previews without the full complexity of SSR.

## Hybrid SSR + CSR in Production

Most production apps don't pick one strategy and commit to it across every route. The most common real-world pattern is **SSR for the critical above-the-fold content, CSR for everything else** — split along the seam of "what does the user need to see immediately" vs. "what can load after they're already engaged."

| Phase          | Handler        | Content                                                                            | Timing                                      |
| -------------- | -------------- | ---------------------------------------------------------------------------------- | ------------------------------------------- |
| Initial load   | Server         | Critical content (product name, images, base price, headline copy)                 | Pre-fetched and embedded before HTML ships  |
| FCP            | HTML           | Pre-rendered content visible                                                       | Immediate                                   |
| Hydration      | Client React   | Event listeners attached, components become interactive                            | After JS bundle arrives                     |
| Secondary data | Client `fetch` | Personalized data (inventory, discounted price, recently-viewed, live counters)    | After hydration, while user is reading      |

**Concrete example: an e-commerce product page.** The server pre-fetches and embeds the product name, images, description, and base price — the content that determines whether the page is useful at all. After hydration, the client fetches the inventory count at the user's nearest warehouse, their loyalty-tier discount, the "people viewing this right now" counter, and personalized recommendations.

This works because FCP is determined by when the browser receives content-filled HTML, and the user's attention shifts to that content the moment it appears. Secondary client fetches happen against a backdrop of "user is already reading," so their latency is hidden by the user's own reading time rather than displayed as a loading spinner. Modern frameworks (Next.js, Remix, TanStack Start) treat rendering as a per-route decision, so different pages in the same app can use entirely different strategies — the marketing landing page can be SSG, the product catalog SSR, the authenticated dashboard pure CSR.

## Choosing a Strategy

| Factor             | CSR/SPA   | SSR                    | SSG                             |
| ------------------ | --------- | ---------------------- | ------------------------------- |
| Initial load speed | Slowest   | Fast (usually)         | Fastest                         |
| SEO                | Poor      | Good                   | Good                            |
| Dynamic content    | Excellent | Good                   | Build-time only (ISR mitigates) |
| Interaction speed  | Excellent | Good (after hydration) | Good (after hydration)          |
| Hosting cost       | Lowest    | Higher                 | Lowest                          |
| Complexity         | Lowest    | Highest                | Medium                          |
| TTI gap            | None      | Can be significant     | Can be significant              |

### Decision Framework

A practical sequence of questions to settle on a strategy:

1. **Does the content change per user?** If yes, you need SSR or CSR — SSG cannot personalize. If no, SSG is on the table.
2. **Does SEO matter?** If yes, the server must produce a content-filled HTML response on the first request — that means SSR, SSG, or RSC. If no (authenticated-only apps, internal tools), SPA/CSR is fine.
3. **How dynamic is the content?** Real-time → SPA or SSR with client updates. Occasional updates → SSG with ISR. Truly static → SSG.
4. **Is JavaScript bundle size a critical constraint?** If yes, RSC is the strongest answer because server-only code never reaches the browser. If no, any of the others.
5. **What infrastructure can you run?** Static-only hosting → SSG or SPA. Server runtime available → SSR or RSC.

A useful sanity check on the answer: many production apps at large organizations (banks, internal SaaS, design tools) ship as plain SPAs not because the team didn't know about SSR, but because their actual users are authenticated, SEO doesn't apply, and interaction performance dominates initial load. SSR/SSG/RSC excel for public, search-dependent, content-heavy pages; SPAs dominate authenticated, interaction-heavy tools.

The right choice depends on your users, your content, and what you measure. There are no silver bullets.

## See Also

- [Web Performance Metrics](web-performance-metrics.md)
- [Bundle Size Optimization](bundle-size-optimization.md)
- [React Server Components](react-server-components.md)
- [Lazy Loading and Suspense](lazy-loading-and-suspense.md)
- [Interaction Performance](interaction-performance.md)
