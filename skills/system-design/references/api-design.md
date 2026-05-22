# API Design

The contract between clients and your system. A good API is predictable, hard to misuse, and stable over time. This file covers the transport foundations, the protocol choices, and the practices that make an API pleasant to consume.

## Contents
- Transport layer: TCP vs UDP
- API protocols overview
- REST
- GraphQL
- REST vs GraphQL (how to choose)
- API design practices (versioning, pagination, errors, idempotency, rate limiting)

## Transport layer: TCP vs UDP

Every API ultimately rides on one of these. Knowing the difference explains a lot of higher-level behavior.

- **TCP** — connection-oriented and reliable. It establishes a connection (the three-way handshake: SYN → SYN-ACK → ACK), guarantees that bytes arrive **in order** and **without loss** (retransmitting what's dropped), and does flow/congestion control. The cost is overhead and latency from setup and acknowledgements. Use it when every byte matters: web (HTTP/HTTPS), database connections, file transfer, email.
- **UDP** — connectionless and unreliable. Fire-and-forget datagrams: no handshake, no ordering or delivery guarantees, no retransmission. Minimal overhead and lowest latency. Use it when speed beats perfection and the application tolerates loss: live video/voice, gaming, DNS lookups, telemetry.

Rule of thumb: **TCP for correctness, UDP for real-time speed.** (HTTP/3 is a notable twist — it runs over QUIC, which is built on UDP but re-adds reliability and ordering at the application layer to avoid TCP's head-of-line blocking.)

## API protocols overview

The common ways services expose functionality, and where each fits:

- **REST** — resource-oriented over HTTP. The default for public web APIs: simple, cacheable, ubiquitous tooling.
- **GraphQL** — a query language where the client specifies exactly the data it wants from a single endpoint. Great for varied clients and complex/nested data.
- **gRPC** — high-performance RPC using Protocol Buffers over HTTP/2. Excellent for internal **service-to-service** communication: fast, strongly-typed, streaming-capable. Less browser-friendly.
- **WebSockets** — a persistent, bidirectional connection for real-time push (chat, live dashboards, multiplayer). Use when the server must push to the client, not just respond.
- **Webhooks** — the server calls *you* (an HTTP callback) when an event happens, instead of you polling. Good for event notifications across systems.

Pick by communication pattern: request/response public API → REST or GraphQL; internal microservices → gRPC; real-time bidirectional → WebSockets; event notifications → webhooks.

## REST

REST models everything as **resources** identified by URLs, manipulated with standard HTTP methods. It's an architectural style, not a strict spec, but good REST APIs share these properties:

- **Resource-oriented URLs** name nouns, not actions: `/users/123/orders`, not `/getUserOrders`.
- **HTTP methods carry the verb**: `GET` (read, safe), `POST` (create), `PUT` (replace), `PATCH` (partial update), `DELETE` (remove).
- **Statelessness**: each request carries everything needed to serve it; the server keeps no client session state between calls. This is what lets any server behind a load balancer handle any request.
- **Meaningful status codes**: `2xx` success, `4xx` client error (`400` bad request, `401` unauthenticated, `403` unauthorized, `404` not found, `409` conflict, `429` too many requests), `5xx` server error.
- **Cacheability**: `GET`s can be cached (by clients, CDNs, proxies), which is a major scaling lever REST gets nearly for free.

Honor method semantics: `GET` must never mutate state; `PUT` and `DELETE` should be **idempotent** (calling twice has the same effect as once).

REST's known weaknesses are **over-fetching** (an endpoint returns more than a given client needs) and **under-fetching** (a client must hit several endpoints to assemble one screen, the "N+1 requests" problem). GraphQL exists largely to solve these.

## GraphQL

A single endpoint where the client sends a **query describing exactly the shape of data it wants**, and gets back exactly that — no more, no less.

- **Strengths**: eliminates over- and under-fetching (one round trip gets precisely the needed fields, even across nested relationships); a strongly-typed schema serves as living documentation and contract; one endpoint serves many different clients (web, mobile, third parties) without bespoke endpoints for each.
- **Costs and risks**: HTTP caching is harder (everything is a `POST` to one URL, so you lean on application-level caching like persisted queries / dataloaders); a naive resolver causes **N+1 database queries** (mitigate with batching/DataLoader); clients can craft expensive deep/nested queries, so you need **query depth/complexity limits** and cost analysis to prevent abuse; more server-side machinery than plain REST.

## REST vs GraphQL (how to choose)

There's no universal winner — match it to the clients and data:

- **Choose REST** when: the API is simple or resource-centric, you want HTTP caching and CDN-friendliness for free, consumers are uniform, or you value operational simplicity. It's the safe default.
- **Choose GraphQL** when: you have **many heterogeneous clients** with different data needs, the data is deeply **nested/relational**, mobile clients are bandwidth-sensitive, or over-/under-fetching is a real, measured pain.
- It's also valid to **mix**: GraphQL for the product-facing API, REST/gRPC internally.

State the tradeoff explicitly when recommending one — the cost of GraphQL is caching complexity and abuse protection; the cost of REST is endpoint proliferation and fetching inefficiency.

## API design practices

These apply regardless of protocol and separate a toy API from a production one.

- **Versioning**: APIs evolve; never break existing clients. Version via URL path (`/v1/...`), header, or content negotiation. Plan for it from day one — retrofitting versioning is painful.
- **Pagination**: never return an unbounded list. **Offset/limit** is simple but degrades on deep pages and can skip/duplicate rows under concurrent writes. **Cursor-based** (keyset) pagination is stable and scales — prefer it for large or fast-changing datasets.
- **Filtering, sorting, field selection**: let clients ask for what they need via query params, reducing payloads and round trips.
- **Consistent errors**: return a structured error body (machine-readable code + human message) with the right status code. Consistency lets clients handle errors generically.
- **Idempotency**: make retries safe. `GET`/`PUT`/`DELETE` are naturally idempotent; for `POST`, accept an **idempotency key** so a client retrying after a timeout doesn't create duplicates (essential for payments and order creation).
- **Rate limiting**: protect the system from abuse and runaway clients. Common algorithms: **token bucket** (allows bursts up to a cap, refills steadily — the usual default), **leaky bucket** (smooths to a fixed rate), **fixed/sliding window** counters. Communicate limits with `429 Too Many Requests` and `Retry-After` / `RateLimit-*` headers so clients can back off gracefully.
- **Authentication & authorization**: covered in `auth-and-security.md` — but decide it as part of API design, not after.
