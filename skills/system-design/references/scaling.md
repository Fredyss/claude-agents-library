# Scaling

How a system grows from one box to a distributed architecture. The throughline: each new piece exists to relieve a *specific* limit. Add nothing speculatively.

## Contents
- The single-server baseline
- Vertical vs horizontal scaling
- Stateless services (the prerequisite for scaling out)
- Database scaling: replication and partitioning
- Caching
- CDNs
- Asynchronous processing with message queues
- Single points of failure (SPOF)

## The single-server baseline

Every design starts here: clients → one application server → one database, all on one machine or a small set. It's cheap, simple, and easy to reason about. It is also the thing you scale *away from*, one bottleneck at a time.

When this baseline strains, the symptom tells you what to fix next:
- CPU/RAM saturated on the app server → scale the app tier (vertical first, then horizontal).
- Database read latency climbing → add a cache, then read replicas.
- Database can't hold the data or absorb the writes → partition/shard.
- Slow asset delivery to distant users → CDN.

Resist fixing limits you haven't hit yet. A single modern server handles a surprising amount of traffic.

## Vertical vs horizontal scaling

**Vertical scaling (scale up)** — give one machine more CPU, RAM, or faster disks. Simple, no code changes, no distribution problems. But there is a hard ceiling (you can only buy so big), it gets expensive non-linearly, and the machine remains a single point of failure. Good first move; bad final answer.

**Horizontal scaling (scale out)** — add more machines and spread load across them. Effectively unlimited headroom and natural redundancy (lose one node, the rest carry on). The cost is complexity: you now need a load balancer, the servers must be stateless, and any shared state (sessions, data) becomes a coordination problem.

Rule of thumb: scale vertically until it's no longer cost-effective or you need redundancy, then scale horizontally. Most serious systems end up horizontal because availability — not just throughput — demands more than one machine.

## Stateless services

Horizontal scaling only works if any server can handle any request. That requires the application tier to be **stateless** — no per-user data living in a single server's memory between requests.

Move state out of the app servers into shared infrastructure:
- Session data → a shared store (e.g. Redis) or a self-contained token (see `auth-and-security.md`).
- Uploaded files → object storage (e.g. S3), not local disk.
- Anything else server-specific → externalize it.

Once servers are stateless, a load balancer can route a user's requests to any node, you can add/remove nodes freely, and a dead node loses nothing. Statelessness is the unlock for almost everything else in this file.

## Database scaling

The database is usually the first hard bottleneck because it's the one component that's hardest to simply duplicate (it holds state). Two distinct levers:

### Replication (for read scaling and availability)
Keep copies of the data on multiple nodes. The common pattern is **primary-replica** (a.k.a. leader-follower): writes go to the primary, which streams changes to read replicas; reads can be served by any replica.
- Solves: read-heavy load, and availability (promote a replica if the primary dies).
- Cost: **replication lag** — replicas are slightly behind, so a read right after a write may see stale data (eventual consistency). Route reads that must be fresh to the primary, or read-your-own-writes from the primary.
- Does *not* solve write throughput (all writes still hit one primary) or total data volume.

### Partitioning / sharding (for write and storage scaling)
Split the data itself across multiple nodes, each owning a subset.
- **Vertical partitioning**: different tables/columns on different nodes (split by feature).
- **Horizontal partitioning (sharding)**: same schema, rows split by a **shard key** (e.g. user_id range or hash).
- Solves: datasets and write volume too large for one machine.
- Cost: this is the most operationally expensive move in system design. Cross-shard queries and joins become hard or impossible; transactions across shards are painful; choosing a bad shard key creates **hot shards** (uneven load); rebalancing is a project. **Delay sharding as long as replication + caching + a bigger box can carry you.**

Decision order under growing load: bigger box → cache → read replicas → shard (last resort).

## Caching

A cache stores the results of expensive operations so repeat requests are served fast from memory instead of recomputing or re-querying. The single highest-leverage performance tool, because most workloads are read-heavy and repetitive.

Where caches live: in the client, at a CDN edge, in a dedicated tier (Redis/Memcached) between app and DB, or inside the database itself.

**Cache-aside (lazy loading)** is the most common pattern: app checks cache → on a miss, reads the DB and populates the cache → returns. Simple and resilient (a cache outage just means slower reads, not failure).

The hard parts of caching are **invalidation** and **staleness** — a cache only helps if reads repeat and some staleness is tolerable:
- **TTL (time to live)**: entries expire after N seconds. Simple; data can be stale up to the TTL.
- **Write-through / write-around / write-back**: different ways to keep cache and DB in sync on writes, trading consistency against write latency.
- **Eviction** (LRU, LFU): when the cache is full, what gets dropped.

Only cache data that is read far more than written and where brief staleness is acceptable. Caching frequently-changing, rarely-read data adds complexity for nothing.

## CDNs (Content Delivery Networks)

A geographically distributed network of edge servers that cache content close to users. A request is served from the nearest edge instead of crossing the planet to your origin.
- Best for static and cacheable assets: images, video, JS/CSS, downloads — and increasingly cacheable API responses at the edge.
- Wins: lower latency for distant users, massive offload from your origin servers, absorbs traffic spikes and some DDoS.
- Same staleness concern as any cache — control it with cache headers (TTL) and explicit invalidation/purge on deploy.

## Asynchronous processing with message queues

Not every unit of work must finish inside the request/response cycle. A **message queue** (e.g. RabbitMQ, Kafka, SQS) lets a producer hand off a task and return immediately while a separate pool of workers processes it.
- Use for: sending email/notifications, image/video processing, generating reports — anything the user doesn't need to wait for.
- Wins: snappy responses, **decoupling** (producers and consumers scale independently), and a **buffer** that absorbs spikes so a traffic surge fills the queue instead of crashing the system.
- Cost: eventual consistency (the work happens "later"), and you must handle retries, ordering, and idempotency (a message may be delivered more than once).

## Single points of failure (SPOF)

A SPOF is any component whose failure brings down the whole system. The single most important reliability exercise is to walk the architecture and ask of each box: *"if this dies right now, what happens?"* Any answer of "everything stops" is a SPOF to eliminate.

Common SPOFs and their fixes:
- **One app server** → multiple servers behind a load balancer.
- **One database** → replication with automatic failover (promote a replica).
- **The load balancer itself** → run it redundantly (active-passive pair with a floating IP, or a managed/anycast LB).
- **One availability zone / data center** → deploy across multiple AZs or regions.
- **One cache holding critical state** → cluster it, and design so a cache miss degrades gracefully (slower) rather than failing.

The general remedy is **redundancy** (no component runs alone) plus **automatic failover** (the system detects death and reroutes without a human). Redundancy without failover just means you find out about the outage faster. See `load-balancing.md` for how health checks drive failover.
