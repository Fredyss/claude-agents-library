---
name: system-design
description: Reason about scalable backend architecture and answer system design questions across databases, scaling, load balancing, APIs, and security. Use this skill whenever the user is designing or reviewing a backend system, asks "how would you design X" (a URL shortener, news feed, chat app, rate limiter, etc.), is preparing for a system design interview, asks how to scale or make a system more reliable, debates SQL vs NoSQL, REST vs GraphQL, or TCP vs UDP, or wants help with load balancing, caching, CDNs, single points of failure, API design, authentication, or authorization — even if they don't say the words "system design" explicitly.
---

# System Design

A practical toolkit for designing scalable, reliable backend systems and answering system design questions. It encodes the standard progression from a single server to a distributed, production-grade architecture, plus the API and security decisions that go with it.

Use this skill to drive a design conversation, evaluate an existing architecture, or answer focused questions ("when do I shard?", "is GraphQL worth it here?"). It is deliberately decision-oriented: most real questions are about *tradeoffs*, so the goal is to help the user pick well for their constraints, not to recite definitions.

## How to use this skill

Most requests fall into one of three modes. Identify the mode first, then proceed.

1. **Design from scratch** ("design a URL shortener", "how would you build a chat app") → run the design workflow below end to end.
2. **Review / scale an existing system** ("we're at 10k req/s and the DB is melting") → skip to the bottleneck the user describes, pull the relevant reference, and propose targeted changes rather than redesigning everything.
3. **Focused question** ("SQL or NoSQL for time-series?", "what's a health check for?") → answer directly from the relevant reference; don't force a full workflow.

Read reference files lazily — only the ones a given request touches. Each reference is self-contained.

| If the request involves… | Read |
|---|---|
| single server, vertical vs horizontal scaling, replication, partitioning/sharding, caching, CDNs, stateless services, SPOF | `references/scaling.md` |
| choosing a data store, SQL vs NoSQL vs graph, consistency, indexing, schema | `references/databases.md` |
| distributing traffic, L4 vs L7, LB algorithms, health checks, removing single points of failure | `references/load-balancing.md` |
| designing endpoints, API protocols, TCP vs UDP, REST, GraphQL, versioning, pagination, rate limiting | `references/api-design.md` |
| login, sessions vs tokens, JWT, OAuth, RBAC/ABAC, common vulnerabilities, securing an API | `references/auth-and-security.md` |

## The design workflow

When designing a system from scratch, walk these steps in order. Narrate the tradeoffs as you go — the *reasoning* is the value, not the final diagram.

1. **Clarify requirements before designing.** Never start drawing boxes from an underspecified prompt. Pin down:
   - *Functional*: what must the system actually do? (the 2-3 core features, not every feature)
   - *Non-functional*: expected scale (DAU, reads/writes per second, data volume), read-vs-write ratio, latency targets, consistency vs availability needs, and the read/write access patterns.
   - State your assumptions explicitly so the user can correct them. A back-of-envelope estimate (e.g. "100M users × 10 reads/day ≈ 12k reads/s average, ~5× peak") anchors every later decision.

2. **Start with the simplest thing that could work.** Sketch the single-server version: one app server, one database, talking to clients. This is the baseline. Everything that follows is justified by a *specific* limit this baseline hits — don't add components speculatively. See `references/scaling.md`.

3. **Choose the data layer.** Pick the store(s) from the access patterns identified in step 1, not from habit. Decide SQL vs NoSQL vs graph, and how you'll handle reads at scale (replication) and data volume (partitioning). See `references/databases.md`.

4. **Define the API.** Settle the contract between clients and the system: protocol (REST/GraphQL/gRPC), resource modeling, pagination, error handling, idempotency, and rate limiting. See `references/api-design.md`.

5. **Scale out and remove single points of failure.** Introduce horizontal scaling, a load balancer, caching, and a CDN *as each is justified* by the numbers from step 1. Then hunt for SPOFs — any one component whose failure takes down the system — and add redundancy. See `references/load-balancing.md` and `references/scaling.md`.

6. **Secure it.** Layer in authentication, authorization, and transport/data security. Security is not a final bolt-on; flag it as decisions are made (e.g. tokens in step 4). See `references/auth-and-security.md`.

7. **Name the tradeoffs and bottlenecks.** Close by stating what the design optimizes for, what it sacrifices, and where it will break next as load grows. A design with no acknowledged tradeoffs is a design that hasn't been thought through.

## Operating principles

- **Tradeoffs over absolutes.** There is rarely a single right answer. "It depends — here's on what" is usually the correct framing. Tie every recommendation to the requirements from step 1.
- **Justify each component by a constraint it relieves.** If you can't name the limit a cache, queue, or shard is solving, don't add it. Premature distribution is a classic failure mode.
- **Estimate, don't hand-wave.** Rough numbers (QPS, storage, bandwidth) turn vague debates into clear decisions. Show the arithmetic.
- **Prefer boring, proven technology** unless a requirement genuinely demands something exotic.
- **Match depth to the audience.** For interview prep, emphasize structured reasoning and the breadth of options considered. For a real architecture decision, go deeper on the specific constraint and the operational cost of each option.

## Quick reference: when each tool earns its place

This is a fast lookup for "do I even need X here?" — full reasoning lives in the reference files.

- **Replica databases** — read-heavy load the primary can't serve alone. (databases / scaling)
- **Sharding / partitioning** — dataset or write volume exceeds one machine. Adds major operational complexity; delay it. (databases / scaling)
- **Cache** — the same expensive reads repeat often and slight staleness is acceptable. (scaling)
- **CDN** — static or cacheable assets served to a geographically spread audience. (scaling)
- **Load balancer** — more than one app server, or you want zero-downtime deploys and health-based routing. (load-balancing)
- **Message queue** — work can be done asynchronously, or you need to absorb spikes and decouple producers from consumers. (scaling)
- **GraphQL** — diverse clients with varying data needs and over/under-fetching is a real pain. Otherwise REST is simpler. (api-design)
