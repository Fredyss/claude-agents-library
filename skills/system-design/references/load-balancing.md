# Load Balancing & Health Checks

A load balancer (LB) sits in front of a pool of servers and distributes incoming requests across them. It is the component that makes horizontal scaling usable and is central to both **performance** (spread the load) and **availability** (route around dead servers).

## Contents
- What a load balancer buys you
- Layer 4 vs Layer 7
- Load balancing algorithms
- Sticky sessions
- Health checks
- The load balancer as a SPOF

## What a load balancer buys you

- **Distribution**: no single server is overwhelmed while others sit idle.
- **High availability**: it stops sending traffic to a server that fails its health check, so one dead node doesn't produce user-facing errors.
- **Zero-downtime operations**: drain a server out of the pool to deploy or patch it, then add it back — users never notice.
- **Elasticity**: add or remove servers behind the LB to match demand.
- **A clean seam** for TLS termination, request routing, and basic rate limiting.

It only works if the app tier is **stateless** (see `scaling.md`) — otherwise routing a user to a different server loses their context.

## Layer 4 vs Layer 7

Load balancers operate at different layers of the network stack, and the choice affects what they can do:

- **Layer 4 (transport)** balances on IP and TCP/UDP port without inspecting the payload. It just forwards packets/connections. Very fast and protocol-agnostic, but it can't make decisions based on content (can't route by URL path or read cookies).
- **Layer 7 (application)** understands HTTP. It can route by path or host (`/api` → API servers, `/img` → image servers), terminate TLS, inspect headers/cookies, retry failed requests, and do content-based rules. More capable, slightly more overhead.

Default to **L7** for web/HTTP services — the routing flexibility is usually worth it. Use **L4** when you need raw throughput, non-HTTP protocols, or the LB shouldn't see decrypted traffic.

## Load balancing algorithms

How the LB picks which server gets the next request:

- **Round robin**: rotate through servers in order. Simple; assumes requests are roughly equal in cost and servers are roughly equal in capacity.
- **Weighted round robin**: like round robin but bigger/faster servers get proportionally more traffic. Good for heterogeneous fleets.
- **Least connections**: send to the server with the fewest active connections. Better when request durations vary a lot (some requests are long-lived), because it tracks actual load rather than assuming uniformity.
- **Least response time**: combines active connections with measured latency — route to the server that's both free and fast.
- **IP hash**: hash the client IP to consistently map a client to the same server. A way to get session affinity without cookies, but rebalances poorly when the server set changes.

Start with round robin (or least-connections if request costs vary), and only get fancier if you observe imbalance.

## Sticky sessions

"Stickiness" pins a given client to the same backend server (via a cookie or IP hash) so that server-local state stays valid across their requests.

Treat stickiness as a **workaround, not a goal.** It undermines even load distribution and means a server's death loses its users' sessions. The better fix is to make the app stateless and externalize session state (shared cache or a self-contained token), so any server can serve any request and stickiness becomes unnecessary.

## Health checks

A health check is the LB periodically probing each backend to decide whether it should receive traffic. This is the mechanism that turns "we have redundancy" into "we have automatic failover."

- **Passive**: infer health from real traffic (count errors/timeouts on actual requests). Cheap but reactive.
- **Active**: the LB sends a dedicated probe on a schedule (e.g. `GET /health` every few seconds) and acts on the response.

Design the health endpoint thoughtfully:
- **Shallow (liveness)**: "is the process up and responding?" — a plain `200 OK`. Fast, but a server can pass this while being unable to do real work.
- **Deep (readiness)**: "can this server actually serve requests?" — verify critical dependencies (database reachable, cache reachable) before reporting healthy. More accurate, but be careful: if every server's deep check depends on one shared DB and that DB blips, *all* servers can fail health checks at once and the LB removes the entire fleet. Tune thresholds (N consecutive failures before ejecting; N successes before re-adding) to avoid flapping.

A server that fails its check is pulled from rotation; when it recovers and passes again, it's added back — no human in the loop. That automatic loop is what protects availability.

## The load balancer as a SPOF

If you put one load balancer in front of everything, you've just moved the single point of failure to the load balancer. Eliminate it the same way you eliminate any SPOF — with redundancy:
- **Active-passive**: a standby LB takes over a shared/floating IP via failover if the primary dies.
- **Active-active**: multiple LBs share traffic, often fronted by DNS round-robin or anycast.
- In practice, a **managed/cloud load balancer** handles this redundancy for you and is the pragmatic default.

See `scaling.md` for the broader treatment of single points of failure.
