# React Rendering Strategies — upskills.dev

> Source: [upskills.dev — React Rendering Strategies](https://upskills.dev/tutorials/react-rendering-strategies?sections=1,2,3,4,5,6)
> Author: Vu Nguyen, Lead Engineer at NAB
> Published: 2026-03-02
> Series: Indie Dev Toolkit, Part 1 of 9

> Note: This is structured notes extracted from the article (the source is copyrighted; full verbatim reproduction was declined by the fetch tool). Quoted phrases are < 125 chars each.

## Section 1 — Evolution of React Rendering

**Pre-2010: Server-Rendered Model.** ".NET MVC developers" wrote business logic in controllers, HTML in Razor views. Every interaction triggered a "full round-trip to the server." Pain points: full page reloads for every interaction, limited interactivity, HTML generation tightly coupled with business logic.

**Late 2000s: jQuery Era.** Middle ground: "Keep the server in charge of rendering HTML, but sprinkle in jQuery." Patterns like `$.ajax()` for forms and `$('.panel').slideToggle()` for UI. The problem: trying to "build applications with a tool designed for enhancements."

**2010–2013: First Wave of SPAs.** Backbone.js (2010), Knockout.js (2010), AngularJS (2010), Ember.js (2011). Introduced declarative data bindings and client-side routing. Shared problems: complex state management, unpredictable DOM updates, two-way binding "made it easy to build small demos but nightmarish to debug at scale."

**2013: React's Breakthrough.** Different philosophy: "state changes → React re-renders → only the changed parts of the DOM update." Solved the right problems and won market share.

**Post-2016: Return to Server.** SPA trade-offs surfaced: SEO challenges, slow initial load, loading spinners, bundle bloat, waterfall requests. Community started bringing server rendering back.

Key takeaway: every pattern emerged because the previous approach hit a real production constraint.

---

## Section 2 — Client-Side Rendering / SPA

Empty HTML shell + JS bundle. Browser downloads, parses, and executes before any rendering occurs.

**Performance profile:**
- Fast after initial load (cost amortized across many interactions).
- Slow initial paint dependent on bundle size.
- Data fetching begins only after React mounts.

**Strengths:** excellent DX (hot module reload), fast subsequent navigation (typically 50–200 ms), rich interactive experiences (offline-first, optimistic updates, persistent state), simplified server infrastructure (just static file serving).

**Weaknesses:** slow first contentful paint, SEO challenges (crawlers may not index fully), bundle bloat as apps grow, full dependency on JavaScript execution.

**Best for:** admin dashboards, design tools, real-time collaboration apps, internal platforms where SEO is irrelevant.

---

## Section 3 — Server-Side Rendering (SSR)

Server fetches data and renders complete HTML before sending to browser. React then "hydrates" the pre-rendered DOM on the client side.

**Performance profile:**
- Higher TTFB (server processing required per request).
- Faster FCP (real content in HTML).
- Faster LCP (no client data fetch needed).
- Gap between FCP and TTI while hydration occurs ("uncanny valley" — visible but not interactive).

**Strengths:** reliable SEO indexing, correct social media previews, no client-side data waterfalls, content available immediately.

**Weaknesses:** server compute required per request, infrastructure/deployment complexity, the FCP→TTI uncanny valley, higher infrastructure costs at scale.

**Best for:** e-commerce product pages, news/media sites, user-specific public content, search result pages.

---

## Section 4 — Static Site Generation (SSG)

Rendering work happens at build time → pre-rendered HTML files deployed to a CDN. No server processing per request.

**Performance profile:**
- Fastest TTFB (CDN serves pre-built files).
- Fastest FCP and LCP.
- TTI still requires hydration but benefits from faster TTFB.

### Incremental Static Regeneration (ISR)

The "stale-while-revalidate" pattern: "when a cached page is old enough to warrant refreshing, serve the stale version immediately and kick off a background re-generation in parallel."

Flow:
1. Current user receives slightly outdated page instantly.
2. Background regeneration runs in parallel.
3. Next user receives the freshly-generated content.

Next.js Pages Router:
```javascript
export async function getStaticProps() {
  return {
    props: { data },
    revalidate: 3600  // revalidate every hour
  }
}
```

Next.js App Router:
```javascript
const data = await fetch(url, { next: { revalidate: 3600 } })
```

Benefit: "Gives you SSG's performance profile (CDN-served files, zero per-request compute) with content that stays reasonably fresh."

**Strengths:** zero infra management, unlimited scalability, lowest operational cost, global CDN distribution by default.

**Weaknesses:** build time grows with content volume, content stale until next deployment (without ISR), no per-request personalization, content updates require rebuild cycles.

**Best for:** marketing pages, documentation, blogs, catalogs (with ISR for fresher content).

---

## Section 5 — React Server Components (RSC)

Granular server/client decisions at the component level rather than per-page.

**Server vs Client Components:**

| Aspect | Server Components | Client Components |
|--------|------------------|-------------------|
| Declaration | Default; can be `async` | Must have `'use client'` at top |
| Runs where | Server only | Server-rendered + hydrated on client |
| Browser bundle | 0 kb | Included |
| State (`useState`) | No | Yes |
| Effects (`useEffect`) | No | Yes |
| Database access | Yes | No |
| Secrets safe | Yes | Exposed to browser |

**`'use client'` boundary:**
- Marks "the entire module subgraph rooted at this file."
- "All transitive imports become client code too."
- Not where the component renders — "where a component's JavaScript ships to the browser."

Server Component example:
```javascript
export default async function Post({ id }) {
  const data = await db.query(id);
  return <article>{data.content}</article>;
}
```

Client Component example:
```javascript
'use client'
export default function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

**Props serialization rules:**
- Allowed: strings, numbers, arrays, plain objects, dates.
- Blocked: functions, class instances, DOM nodes, Map/Set with class values.
- Constraint: "Props passed from Server to Client Components must be serializable."

### Server Functions (`'use server'`)

Mark an async function with `'use server'` to make it "callable from Client Components. The framework serializes the arguments, sends them to the server, executes the function, and returns the result, all automatically."

```javascript
'use server'
export async function savePost(formData) {
  const result = await db.posts.insert(formData);
  return result;
}
```

Called from a Client Component:
```javascript
'use client'
export default function Editor() {
  async function submit(formData) {
    const result = await savePost(formData);
  }
}
```

**Progressive enhancement with forms:**
```javascript
<form action={savePost}>
  <input name="title" required />
  <button type="submit">Save</button>
</form>
```

The form "works with or without JavaScript, with progressive enhancement built in."

**Security note:** "Treat every Server Function argument as untrusted input… Validate the input, verify the caller's authorization." Every server function is essentially a public API endpoint.

### Streaming with Suspense

"Server Components stream progressively. Wrap slow fetches in Suspense and fast content arrives first."

How it works:
- Multiple data fetches run in parallel on server.
- Fastest component resolves first; its HTML streams immediately.
- Suspense boundary shows fallback while data loads.
- "No client-side refetching needed. No loading state code in any of the leaf components."

```javascript
export default function Page() {
  return (
    <>
      <Suspense fallback={<Skeleton />}>
        <FastComponent />
      </Suspense>
      <Suspense fallback={<Skeleton />}>
        <SlowComponent />
      </Suspense>
    </>
  );
}
```

Benefit: "Data requirement lives right next to the component that uses it" — no prop drilling from centralized data fetchers.

**Strengths:** dramatically smaller JS bundles, co-located data fetching, direct infra access (DBs, secrets stay private), streaming with Suspense for progressive rendering.

**Weaknesses:** new mental model, framework dependency essentially required, library ecosystem still adapting, props crossing boundaries must serialize, debugging spans two environments.

---

## Section 6 — Patterns in the Real World

Production apps typically combine strategies. Most common hybrid: **SSR + CSR**.

### SSR + CSR Hybrid

| Phase | Handler | Content | Timing |
|-------|---------|---------|--------|
| Initial load | Server | Critical data (product name, images, base price) | Prefetch before HTML ships |
| FCP | HTML | Pre-fetched content fills page | Immediate |
| Hydration | Client React | Event listeners attached | After JS bundle arrives |
| Secondary data | Client fetch | Personalized data (inventory, discounted price, real-time counters) | After hydration |

**E-commerce product page example.** Server pre-fetches: "product name, images, description, and base price." After hydration, client fetches: "inventory count at the user's nearest warehouse, their loyalty-discounted price, recently viewed items, the live 'people viewing this' counter."

Why it works:
- "First Contentful Paint is determined by when the browser receives content-filled HTML."
- Secondary client fetches "happen after the user is already reading content."
- "Perceived latency drops dramatically because the user's attention is on the page they can already see, not a loading indicator."

### Route-Level Mixing

"Most applications don't commit to a single pattern. They combine them." Modern frameworks "treat rendering as a per-route decision, not a per-app one." Example combinations:
- SSG + CSR: static landing pages with interactive widgets.
- SSG + ISR: documentation rebuilt nightly.
- RSC + Client Components: server-rendered data tables with client-side sorting.
- SSR + RSC: complex per-request apps.

### Decision Framework

Primary questions:
1. **Does content change per user?** Yes → SSR (per-request personalization). No → SSG (identical for everyone).
2. **Does SEO matter?** Yes → SSR or SSG (full HTML needed). No → SPA possible (auth-only apps).
3. **How dynamic is content?** Real-time → SPA or SSR with client updates. Occasional updates → SSG with ISR. Build-time static → SSG.
4. **Bundle size critical?** Yes → RSC (server code stays server-side). No → SPA acceptable.
5. **Infrastructure constraints?** Static-only → SSG/SPA. Server required → SSR/RSC.

Overarching principle: "The right rendering pattern isn't the most sophisticated or the most fashionable. It's the one that delivers the best possible experience to the specific users you're building for."

The author notes that many production applications at major organizations (including NAB) use SPAs not from ignorance but because they solve the actual problems those applications face. SSR and SSG excel for public-facing, search-dependent content; SPAs dominate authenticated, interaction-heavy tools where SEO doesn't apply.

---

## Comparison Summary

| Pattern | Best for | TTFB | FCP | Bundle | Server load |
|---------|----------|------|-----|--------|-------------|
| SPA | Authenticated, interaction-heavy | Fast | Slow | Large | None |
| SSR | Dynamic, SEO-critical, per-request | Slower | Fast | Large | Per-request |
| SSG | Static, identical for all users | Fastest | Fastest | Large | None (CDN) |
| RSC | Data-heavy, server-fetching | Slower | Fast | Smallest | Per-request |

---

## Frameworks Mentioned

**Next.js (primary focus).**
- Pages Router (legacy): `getServerSideProps` runs per request (SSR); `getStaticProps` runs at build time (SSG). Both look "almost identical, but the timing is completely different."
- App Router (modern): direct `fetch` in components — "you can just fetch directly in the component body, no explicit data-fetching function needed." RSC is the default; add `'use client'` for interactivity.

**Other frameworks named:** Vercel/Netlify (static deployment), Remix and TanStack Start (deferred to Part 2 of the series).

What frameworks enable: "Zero-Config Scaffolding: from npm create to a running dev server with HMR, TypeScript, and ESLint, all in under 60 seconds."

---

## Key Takeaways

1. Each rendering pattern solved genuine production problems teams encountered.
2. Trade-offs between patterns remain stable despite framework evolution.
3. Bundle size and data-fetching strategy are the primary performance levers.
4. No single pattern is universally correct — context determines the right choice.
5. Modern frameworks enable mixing strategies per-route within a single codebase.
