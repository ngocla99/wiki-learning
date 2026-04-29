# Database Selection Factors

> Sources: ByteByteGo (Alex Xu), 2023-04-26
> Raw: [Factors to Consider in Database Selection](../../raw/system-design/2023-04-26-factors-database-selection-bytebytego.md)
> Updated: 2026-04-29

## Overview

Once you understand [the major database types](database-types.md), choosing the right one for a workload comes down to seven factors. Each factor exposes a different tradeoff — and most real applications mix databases because no single engine optimizes for all seven simultaneously.
![[Pasted image 20260429145052.png]]
## 1. Scalability

Two axes: **vertical** (bigger box — more CPU/RAM) and **horizontal** (more boxes — sharding/partitioning).

| Type        | Scaling story                                                                         |
| ----------- | ------------------------------------------------------------------------------------- |
| Relational  | Vertical scales naturally; horizontal is hard because cross-node consistency is costly |
| NoSQL       | Built for horizontal scale via sharding/partitioning — at the cost of strict consistency |
| NewSQL      | Distributed architecture preserves SQL/ACID while scaling out — younger, more complex   |
| Time-series | Horizontal scale tuned for high-velocity ingestion via specialized indexing/compression |
![[download 8.webp]]
**Decision lens:** Project growth curve. If you're sure of vertical-only scale, traditional SQL is simpler. If horizontal scale is mandatory, accept either eventual consistency (NoSQL) or operational complexity (NewSQL).

## 2. Performance

Performance has two dimensions: **query efficiency** (especially complex joins/aggregations) and the **read/write balance**.

- **Relational** — strong on complex queries thanks to SQL and structured schema; degrades on very large datasets.
- **NoSQL** — fast writes from simple data models; weaker on complex queries and aggregations.
- **NewSQL** — distributed query processing + advanced indexing aim to give both, but with operational cost.
- **Time-series** — purpose-built for time-range queries on append-heavy workloads.

**Decision lens:** Profile your workload. Read-heavy analytics ≠ write-heavy ingestion ≠ low-latency point lookups — each prefers a different engine.

## 3. Data Consistency

Consistency models flow from two theorems:

- **CAP theorem** — under partition, choose between **Consistency** and **Availability**. (You don't really get to drop Partition tolerance in distributed systems.)
- **PACELC theorem** (better mental model) — *if* **P**artitioned, choose **A**vailability vs **C**onsistency; *else* choose **L**atency vs **C**onsistency. CAP only models the failure case; PACELC also models the steady state.
![[download 9.webp]]
**How databases position themselves:**
- Relational — strong consistency by default (full ACID).
- NoSQL — typically eventual consistency to gain availability/performance.
- NewSQL — strong consistency across distributed nodes (the headline value prop).

**Decision lens:** Financial transactions, inventory, auth → strong consistency. Social feeds, analytics dashboards, search indexes → eventual is usually fine.

## 4. Data Model

Defines how data is structured, stored, and queried. Two sub-questions: **schema flexibility** and **support for complex relationships**.

- **Fixed schema (relational)** — enforces integrity, prevents bad data, but schema migrations are expensive and may require downtime.
- **Flexible/schemaless (NoSQL)** — easy to evolve, natural fit for hierarchical or rapidly changing data; pushes validation responsibility to the application.
- **Relationships** — Relational and Graph excel here. Document/Key-Value databases force you to denormalize or do app-side joins.

**Decision lens:** Match the data model to the *natural shape* of your data. Forcing nested documents into rows (or vice versa) burns engineering time on every read/write.

## 5. Security

Security spans three dimensions:

- **AuthN / AuthZ** — user accounts, roles, RBAC. Mature relational databases lead here.
- **Encryption** — in-transit (TLS) and at-rest (AES-256). Some databases also offer field-level masking/redaction.
- **Ecosystem integration** — compatibility with identity providers, firewalls, IDS/IPS, and the vendor's track record on patching CVEs.
![[download 11.webp]]
**Decision lens:** For regulated workloads (PII, PCI, HIPAA), built-in security primitives + audit logging + a responsive vendor outweigh raw performance.

## 6. Cost

Total Cost of Ownership has two components:

- **Licensing & hosting** — open-source (MySQL, PostgreSQL) is free to license; commercial (Oracle, SQL Server) charges per core/instance. Self-hosted vs cloud-managed shifts cost between staff time and vendor fees.
- **Maintenance & management** — upgrades, staff expertise, tuning, troubleshooting. Operationally complex databases (e.g., Cassandra, distributed NewSQL) carry hidden ongoing costs.

**Decision lens:** Don't optimize licensing fees in isolation — a "free" database that needs a dedicated DBA may cost more than a paid managed service.

## 7. Community and Ecosystem

A strong community and ecosystem reduce time-to-productivity and de-risk operational issues.

- **Developer resources** — quality docs, tutorials, active forums, Stack Overflow coverage.
- **Integrations** — official drivers/ORMs across languages, ETL connectors, observability tooling, BI tool support.
- **Maturity signal** — long-lived projects (PostgreSQL, MongoDB) have battle-tested patterns; newer engines may force you to write your own integrations.

**Decision lens:** A "better" database with poor ecosystem support often loses to a slightly-worse one your team already knows and that integrates cleanly with your stack.

## Summary Checklist

When evaluating a database, walk through these questions in order:

1. **Scale** — Will it survive 10× data and 10× traffic? Vertical or horizontal path?
2. **Performance** — Does the read/write profile match the workload?
3. **Consistency** — Strong, eventual, or tunable? Apply PACELC, not just CAP.
4. **Data model** — Does the natural shape of the data fit?
5. **Security** — AuthN, AuthZ, encryption, audit, vendor patch cadence?
6. **Cost** — License + hosting + ops headcount, not just license.
7. **Community** — Will the team be productive in 6 months without vendor support?

No database scores 10/10 across all seven. The art is choosing **which tradeoffs** are acceptable for **this specific workload** — and accepting that complex applications will mix several engines.
