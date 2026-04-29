# Database Types

> Sources: ByteByteGo (Alex Xu), 2023-04-19
> Raw: [Understanding Database Types](../../raw/system-design/2023-04-19-understanding-database-types-bytebytego.md)
> Updated: 2026-04-29

## Overview

Complex applications typically use several databases in tandem, each serving a specific purpose. Choosing the right type requires understanding the tradeoffs across four primary categories: Relational, NoSQL, NewSQL, and Time-series.
![[download.webp]]

![[download 1.webp]]
## Relational (SQL)

Stores data in tables with rows and columns. The default choice when data is structured and consistency is non-negotiable.

**Strengths**
- ACID transactions (Atomicity, Consistency, Isolation, Durability)
- Powerful SQL queries and joins across related tables
- Referential integrity via primary/foreign keys
- Mature indexing and query optimization

**Weaknesses**
- Horizontal scaling is hard; designed for vertical scale
- Rigid schema — evolving requirements require costly migrations
- Performance degrades with very large datasets and complex queries
- Poor fit for unstructured/semi-structured data

**Examples:** MySQL, PostgreSQL, SQL Server, Oracle
![[download 3.webp]]

---

## NoSQL

Abandons the strict relational model to gain flexibility, scale, or specialized performance. Four distinct subtypes:

### Document Stores

Stores records as JSON/BSON documents with dynamic schemas. Best for hierarchical or nested data.

- **Use cases:** CMS, e-commerce catalogs, user profiles, analytics
- **Examples:** MongoDB, Couchbase

### Column-Family (Wide-Column) Stores

Organizes data by column rather than row. Excellent compression and fast column-scoped reads. Built for distributed, high-throughput workloads.

- **Use cases:** Big data, analytics, high write/read workloads
- **Examples:** Apache Cassandra, HBase

### Key-Value Stores

Simplest model — a hash map at scale. Extremely fast reads/writes, easy horizontal scaling.

- **Use cases:** Caching layers, session stores, real-time leaderboards, config storage
- **Examples:** Redis, Amazon DynamoDB

### Graph Databases

Stores entities as nodes and relationships as edges. Traversing complex relationship networks is first-class.

- **Use cases:** Social graphs, fraud detection, recommendation engines
- **Examples:** Neo4j, Amazon Neptune

**Shared NoSQL weaknesses**
- No standardized query language (each has its own API)
- Eventual consistency by default — strict consistency is an add-on cost
- Limited or absent multi-record transaction support
![[download 4.webp]]

---

## NewSQL

Combines the relational model and ACID guarantees with NoSQL-style horizontal scalability. Targets large-scale, highly concurrent workloads where both consistency and scale matter.

**Key properties**
- Distributed architecture with automatic sharding and replication
- Horizontal scale-out while preserving SQL and ACID
- Advanced concurrency control (MVCC, optimistic locking)
- SQL-compatible — existing tooling often works with minimal changes

**Weaknesses**
- More operationally complex than traditional SQL
- Risk of vendor lock-in (many are managed-only cloud services)
- Less mature ecosystem and community than PostgreSQL/MySQL

**Examples:** CockroachDB, Google Spanner, TiDB

![[download 5.webp]]

---

## Time-Series

Specialized for sequentially ordered, time-stamped data. Ingestion rate and time-range query performance are the primary design axes.

**Key properties**
- Optimized for high-velocity writes (sensor streams, metrics, logs)
- Efficient compression tuned for time-ordered data patterns
- Native retention policies (auto-expire old data)
- Built-in aggregation, downsampling, and forecasting functions
- Horizontal scale for high ingestion rates

**Use cases:** IoT telemetry, infrastructure monitoring, financial tick data

**Examples:** InfluxDB, TimescaleDB
![[download 6.webp]]

---

## Comparison at a Glance

| Type        | Data model         | Scale         | Consistency      | Best for                              |
| ----------- | ------------------ | ------------- | ---------------- | ------------------------------------- |
| Relational  | Tables / rows      | Vertical      | Strong (ACID)    | Structured data, complex queries      |
| Document    | JSON documents     | Horizontal    | Eventual / tunable | Hierarchical, flexible schemas      |
| Column      | Column families    | Horizontal    | Eventual / tunable | Analytics, high write throughput    |
| Key-value   | Key → value        | Horizontal    | Eventual         | Caching, sessions, low-latency reads  |
| Graph       | Nodes + edges      | Limited       | Strong / tunable | Relationship traversal                |
| NewSQL      | Tables / rows      | Horizontal    | Strong (ACID)    | Scale + consistency, global apps      |
| Time-series | Timestamped events | Horizontal    | Strong           | Metrics, IoT, monitoring              |
