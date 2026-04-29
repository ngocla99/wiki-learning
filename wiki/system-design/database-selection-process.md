# Database Selection Process

> Sources: ByteByteGo (Alex Xu), 2023-05-03
> Raw: [Key Steps in the Database Selection Process](../../raw/system-design/2023-05-03-key-steps-database-selection-bytebytego.md)
> Updated: 2026-04-29

## Overview

Once you know [the database types](database-types.md) and [the seven evaluation factors](database-selection-factors.md), the selection process itself is a five-step workflow ending in real-world case studies. The repeated lesson: complex applications mix multiple databases, each chosen for a specific responsibility.

## The Five-Step Process

### 1. Assess Project Requirements

Map the workload before shopping. Capture:

- **Data shape** — structured / semi-structured / unstructured; relationships and patterns the data must preserve.
- **Volume curve** — current size, growth rate, write rate. Is storage or write throughput the bottleneck?
- **Concurrency** — peak concurrent users/connections, expected fluctuation between peak and off-peak.
- **Performance SLAs** — response time, query latency targets, throughput.
- **Compliance** — regulatory requirements (encryption, RBAC, audit logging).
- **Integration surface** — existing systems, frameworks, and tools the database must coexist with.

This list is the rubric for every later step.

### 2. Evaluate Database Options

Build a shortlist of 2–4 candidates, then score each one across the seven factors (scalability, performance, consistency, data model, security, cost, community). Cross-reference user reviews, case studies in similar industries, and vendor track records. Note tradeoffs explicitly — "X wins on writes but loses on joins" — so they're visible in the final comparison.
![[download 12.webp]]
### 3. Performance Test and Benchmark

Build a test environment that mirrors production (same hardware, network, software). Design scenarios that simulate **realistic** workloads: read mix, write mix, complex queries, concurrent connections. Capture metrics for each candidate and create a per-database performance profile.

The point is not to find a "best" number but to surface where each candidate **excels and struggles** relative to the project's actual workload.
![[download 13.webp]]
### 4. Consider Long-Term Implications

Project the decision forward 2–3 years:

- Will it handle 10× data volume? 10× users? New feature surface area?
- What's the **total cost of ownership** — license, hosting, ops headcount, scaling expenses?
- How healthy are the community and the company/foundation backing it? A dying ecosystem is a future migration project.
![[download 14.webp]]
### 5. Make the Final Decision

Re-tabulate the shortlist against the requirements. No database wins on every dimension — the work is **prioritizing which factors are non-negotiable** and which can flex. Pick, deploy, and *keep monitoring*: revisit the choice as the workload evolves.

## Case Studies

Three real-world patterns showing how different workloads drive different choices.

### E-commerce Platform — Hybrid (PostgreSQL + Elasticsearch)
![[download 15.webp]]
**Workload:** Growing product catalog, customer profiles, order history, plus product search and analytics.

**Why a single database failed:** Relational handles transactions well but is weak at full-text search and analytics. A pure NoSQL would have lost ACID guarantees on order/inventory operations.

**Solution:** Split the workload by responsibility.
- **PostgreSQL** owns transactional data — orders, inventory, customers — preserving ACID and rich joins.
- **Elasticsearch** owns search and analytics — fast indexing, real-time search results, recommendations, horizontal scale.

**Lesson:** When workload characteristics diverge sharply (transactional vs analytical), use multiple databases instead of compromising on one.

### Social Media Platform — Amazon DynamoDB (Key-Value)
![[download 16.webp]]
**Workload:** Millions of users, billions of interactions/day. Complex relationships (friendships, comments, shared content). High write throughput, low-latency reads, horizontal scale.

**Why relational failed:** Modeling friendship/comment graphs in a relational schema becomes a join-heavy nightmare at scale.

**Solution:** DynamoDB with **adjacency-list modeling** to represent graph relationships in a key-value store.

| Use case          | Partition key | Sort key                   | Notes                                              |
| ----------------- | ------------- | -------------------------- | -------------------------------------------------- |
| Friendships       | userId        | friendUserId               | Secondary index reverses keys for inverse lookups  |
| User content      | userId        | contentId                  | Filters posts/comments by user                     |
| Post ↔ comment    | userId        | (contentType, contentId)   | Composite sort key — query posts vs comments cleanly |

**Tradeoff:** Massive scale and speed, but the data modeling is unconventional and has a steep learning curve. Engineers must design access patterns *upfront* — DynamoDB punishes ad-hoc queries.

**Lesson:** A key-value store *can* model graph-like data with careful partition/sort key design — pay the modeling cost up front to gain scale and consistent latency.

### IoT Data Processing — InfluxDB (Time-Series)
![[download 17.webp]]
**Workload:** Telemetry from millions of devices. High-velocity ingestion of time-stamped data, real-time aggregation, low-latency analytics, deep historical retention.

**Why a generic database failed:** General-purpose databases waste storage on time-series and lack native windowing/aggregation primitives.

**Solution:** InfluxDB — a purpose-built time-series database.
- **Storage** — specialized columnar/time-series format with strong compression for historical retention.
- **Continuous queries** — pre-aggregate streams in the database, offloading downstream systems.
- **Horizontal scale** — add nodes to the cluster as device count grows.
- **Ecosystem** — mature integrations with monitoring/visualization stacks.

**Lesson:** When the data has a single dominant axis (here, time), a specialized engine beats a general-purpose one on storage cost, ingest rate, and query ergonomics.
![[download 18.webp]]
## Key Takeaways

- **No one-size-fits-all.** The "right" database is a function of *this specific workload*, not a universal ranking.
- **Polyglot persistence is normal.** Modern applications routinely combine 2–4 database types, each owning a slice of the workload.
- **Data modeling matters as much as database choice.** A wrong access-pattern design can sink the right database (DynamoDB case).
- **Decide → monitor → revisit.** Database selection is a checkpoint, not a one-time event. Re-evaluate as the workload evolves.
