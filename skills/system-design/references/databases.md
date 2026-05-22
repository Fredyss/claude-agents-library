# Databases

Choosing and using the right data store. The decision flows from **access patterns and data shape**, not from defaults or fashion. Pick the store that makes your most common and most demanding queries natural and fast.

## Contents
- The three families: relational, NoSQL, graph
- How to choose
- Consistency: ACID vs BASE, and the CAP tradeoff
- Indexing
- Polyglot persistence

## The three families

### Relational (SQL)
Data lives in tables of rows and columns with a fixed, enforced schema and relationships expressed through foreign keys. Queried with SQL. Examples: PostgreSQL, MySQL.

- **Strengths**: strong consistency and **ACID transactions** (critical for money, inventory, anything that must never be partially applied); flexible ad-hoc queries and joins across related data; mature, well-understood, enforced data integrity.
- **Weaknesses**: the rigid schema makes rapid model changes harder; scaling writes horizontally is difficult because joins and transactions resist sharding (though modern "NewSQL"/distributed SQL engines narrow this gap).
- **Reach for it when**: data is relational, you need transactions and integrity, query patterns are varied or not fully known up front. **This is the correct default for most applications** — start here unless you have a concrete reason not to.

### NoSQL
An umbrella for non-relational stores, typically schema-flexible and built to scale horizontally. Four common sub-types:

- **Document** (MongoDB, DynamoDB): stores JSON-like documents. Flexible schema; great when data is naturally nested and usually fetched as a self-contained unit (a user profile, a product).
- **Key-value** (Redis, DynamoDB): a giant hash map. Extremely fast simple lookups; ideal for caching, sessions, feature flags.
- **Wide-column** (Cassandra, HBase): rows with dynamic columns, optimized for huge write volume and time-series/event data across many nodes.
- **Search** (Elasticsearch): inverted indexes for full-text search and analytics.

- **Strengths**: scales out horizontally with relative ease; flexible/evolving schema; high throughput for the access patterns it's designed around.
- **Weaknesses**: usually **eventual** rather than strong consistency; joins are limited or absent (you denormalize and duplicate data instead); you must design the schema *around your queries* up front — get the access pattern wrong and some queries become very expensive.
- **Reach for it when**: massive scale, simple or well-known access patterns, flexible data, and where eventual consistency is acceptable.

### Graph
Data modeled as nodes and edges (relationships are first-class). Examples: Neo4j, Amazon Neptune.

- **Strengths**: traversing relationships ("friends of friends of friends", recommendation paths, fraud rings) is fast and natural — the same query is a punishing chain of joins in SQL.
- **Weaknesses**: niche; overkill unless relationships *are* the core of the problem.
- **Reach for it when**: the relationships between entities matter more than the entities themselves — social graphs, recommendation engines, fraud detection, network/dependency analysis.

## How to choose

Don't ask "which database is best." Ask, in order:

1. **What's the data shape?** Tabular and relational → SQL. Self-contained nested documents → document store. Pure lookups by key → key-value. Heavily interconnected → graph.
2. **What are the access patterns?** Map your top few queries. The store should make *those* trivial. NoSQL especially demands you know these first, because you model around them.
3. **What consistency does the domain require?** Money, inventory, bookings → strong consistency/transactions → SQL. Likes, view counts, feeds → eventual consistency is fine → NoSQL is on the table.
4. **What scale, and which dimension?** Read-heavy is often solved by replication + caching on *any* store. Extreme write volume or data that won't fit one machine pushes toward horizontally-scalable NoSQL or sharded SQL.

When unsure, **default to a relational database.** It's flexible enough for most needs, and you can add specialized stores later for the parts that need them.

## Consistency: ACID vs BASE, and CAP

- **ACID** (Atomicity, Consistency, Isolation, Durability): the transactional guarantees of relational DBs. A transaction fully happens or fully doesn't; the database moves between valid states only. Use when correctness is non-negotiable.
- **BASE** (Basically Available, Soft state, Eventual consistency): the looser model many NoSQL systems adopt to stay available and scalable. Reads may briefly return stale data, converging to correct over time.

**CAP theorem**: in the presence of a network **P**artition, a distributed system must choose between **C**onsistency (every read sees the latest write) and **A**vailability (every request gets a response). You can't have both *during* a partition.
- **CP** systems refuse requests rather than serve stale/inconsistent data (favor correctness — e.g. a banking ledger).
- **AP** systems keep serving, accepting temporary inconsistency (favor uptime — e.g. a social feed).
The practical takeaway: decide per-feature whether stale-but-up or correct-but-maybe-unavailable is the right failure mode, and pick the store accordingly.

## Indexing

An index is a separate data structure (commonly a B-tree) that lets the database find rows without scanning the whole table — like a book's index instead of reading every page. Turns an O(n) scan into roughly O(log n) lookup.
- Add indexes on columns you frequently filter, join, or sort on.
- **The tradeoff**: indexes speed up reads but slow down writes (every insert/update must also update the indexes) and consume storage. Don't index everything — index the queries that matter and measure.
- Composite indexes cover multi-column queries; column order matters and should match query patterns.

## Polyglot persistence

You are not required to pick one database for the whole system. Mature architectures use the right tool per job: PostgreSQL for transactional core data, Redis for caching and sessions, Elasticsearch for search, a wide-column store for the analytics firehose. The cost is operational (more systems to run, more data to keep in sync), so introduce each additional store only when a single store genuinely can't serve a workload well.
