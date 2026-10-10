# System Design Learning Path

A ground-up path from fundamentals to distributed systems, for a Rails developer.
Each phase has theory, then grows one shared project — a ticketing app — so every
concept lands on code that already exists. See [PROJECT.md](PROJECT.md) for the
app spec and how checkpoints work.

**Rule:** Don't start a phase until the previous phase is tagged and you can
explain every decision in it from memory.

**Stack:** Ruby/Rails, PostgreSQL, Redis, Sidekiq, Docker, Nginx.
The point is learning system design concepts, not new languages.

**Primary reading sources:**
- [system-design-primer](https://github.com/donnemartin/system-design-primer) — the foundational reference
- [ByteByteGo system-design-101](https://github.com/ByteByteGoHq/system-design-101) — visual diagrams
- [karanpratapsingh/system-design](https://github.com/karanpratapsingh/system-design) — textbook-style walkthrough

---

## Phase 0 — Checkpoint Zero

**Why this exists:** every later phase is a diff on top of this one. If the
starting point can't be rebuilt from a fresh clone, no checkpoint after it can
either.

### Build
The bare skeleton described in [PROJECT.md → Phase 0](PROJECT.md#phase-0--skeleton):
Rails API app, Postgres, Redis, Docker Compose, schema, deterministic seeds.
No features.

### You're done when:
- [ ] A fresh clone runs with one command and no manual steps
- [ ] Running the seeds twice produces identical data
- [ ] Every version is pinned (Ruby, gems, Docker images)

---

## Phase 1 — How a Request Actually Works

**Why this is first:** every later concept (caching, load balancing, replication)
is a modification to this basic flow. If you can't trace a request end to end,
nothing else has context to attach to.

### Read
- How DNS works
  - [system-design-primer: DNS](https://github.com/donnemartin/system-design-primer#domain-name-system)
  - [karanpratapsingh: DNS](https://github.com/karanpratapsingh/system-design#domain-name-system-dns)
- TCP/IP basics (the transport layer, what a connection actually is)
  - [karanpratapsingh: IP](https://github.com/karanpratapsingh/system-design#ip)
  - [karanpratapsingh: TCP and UDP](https://github.com/karanpratapsingh/system-design#tcp-and-udp)
- HTTP request/response lifecycle
  - [ByteByteGo: HTTP 1.0 → 2.0 → 3.0](https://github.com/ByteByteGoHq/system-design-101#how-does-http-work)
  - [karanpratapsingh: REST, HTTP](https://github.com/karanpratapsingh/system-design#representational-state-transfer-rest)
- How a web server (Puma) receives, routes, and responds
  - [Puma architecture docs](https://github.com/puma/puma/blob/master/docs/architecture.md)

### Project: Request Tracer
[PROJECT.md → Phase 1](PROJECT.md#phase-1--request-tracer)

### You're done when you can:
- [ ] Draw the full request path from client to database and back, naming every layer
- [ ] Explain what happens at each layer if it fails
- [ ] Explain the difference between a connection and a request

---

## Phase 2 — Data Modeling, Databases, and Concurrency

**Why this is second:** before you can talk about scaling a database, you need to
understand what a well-designed one looks like at normal scale — and what happens
when two people write to it at the same moment.

### Read
- ACID properties (what your transactions actually guarantee)
  - [karanpratapsingh: ACID model](https://github.com/karanpratapsingh/system-design#acid-model)
- Normalization vs denormalization (when to break the rules and why)
  - [system-design-primer: Denormalization](https://github.com/donnemartin/system-design-primer#denormalization)
- Indexing: B-tree vs hash, composite indexes, partial indexes
  - [system-design-primer: SQL tuning](https://github.com/donnemartin/system-design-primer#sql-tuning)
  - [Use The Index, Luke (free online book)](https://use-the-index-luke.com/)
- Query planning: how EXPLAIN ANALYZE actually works
  - [PostgreSQL EXPLAIN docs](https://www.postgresql.org/docs/current/using-explain.html)
- N+1 queries: why ORMs create them and the three standard fixes
  - [karanpratapsingh: N+1 query problem](https://github.com/karanpratapsingh/system-design#n1-query-problem)
- Concurrency control: pessimistic vs optimistic locking, `SELECT … FOR UPDATE`,
  isolation levels
  - [PostgreSQL: Explicit locking](https://www.postgresql.org/docs/current/explicit-locking.html)
  - [PostgreSQL: Transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
  - [Rails: Pessimistic locking](https://api.rubyonrails.org/classes/ActiveRecord/Locking/Pessimistic.html)
  - [Rails: Optimistic locking](https://api.rubyonrails.org/classes/ActiveRecord/Locking/Optimistic.html)

### Project: Booking Core + Query Clinic
[PROJECT.md → Phase 2](PROJECT.md#phase-2--booking-core--query-clinic)

### You're done when you can:
- [ ] Read an EXPLAIN ANALYZE output and identify the bottleneck without guessing
- [ ] Explain when denormalization helps and what it costs, using `seats_available` as the example
- [ ] Explain what a composite index on `(event_id, status)` does that two separate indexes wouldn't
- [ ] Explain how you prevent two buyers getting the same seat, and when you'd
      pick pessimistic over optimistic locking

---

## Phase 3 — Caching

**Why this is third:** caching only makes sense once you understand what you're
caching (Phase 2) and how requests flow (Phase 1).

### Read
- Where to cache: client, CDN, application, database query, object
  - [system-design-primer: Cache](https://github.com/donnemartin/system-design-primer#cache)
  - [ByteByteGo: Caching](https://github.com/ByteByteGoHq/system-design-101#caching)
- Cache update patterns: cache-aside, write-through, write-behind, refresh-ahead
  - [system-design-primer: Cache update patterns](https://github.com/donnemartin/system-design-primer#when-to-update-the-cache)
  - [karanpratapsingh: Caching strategies](https://github.com/karanpratapsingh/system-design#caching)
- TTL vs event-driven invalidation
  - [system-design-primer: Cache-aside](https://github.com/donnemartin/system-design-primer#cache-aside)
- Cache stampede / thundering herd problem
  - [karanpratapsingh: Cache invalidation](https://github.com/karanpratapsingh/system-design#cache-invalidation)
- HTTP caching: `Cache-Control`, ETags, conditional GET
  - [Rails guides: Conditional GET](https://guides.rubyonrails.org/caching_with_rails.html#conditional-get-support)
- Redis data structures beyond simple key-value (sorted sets, hashes)
  - [Redis docs: Data types](https://redis.io/docs/data-types/)

### Project: Cache Layer
[PROJECT.md → Phase 3](PROJECT.md#phase-3--cache-layer)

### You're done when you can:
- [ ] Say which cache-update pattern fits event listings vs seat availability, and why they differ
- [ ] Explain what happens to the app if Redis goes down entirely
- [ ] Explain the thundering herd problem and one way to mitigate it
- [ ] Explain TTL-based vs event-driven invalidation, and when each is appropriate

---

## Phase 4 — Background Jobs and Asynchronous Processing

**Why this is fourth:** using Sidekiq daily is different from understanding *why*
a job queue exists and what its failure modes are.

### Read
- Synchronous vs asynchronous processing (when to use which)
  - [system-design-primer: Asynchronism](https://github.com/donnemartin/system-design-primer#asynchronism)
- Message queues vs task queues (what's the actual difference)
  - [system-design-primer: Message queues](https://github.com/donnemartin/system-design-primer#message-queues)
  - [system-design-primer: Task queues](https://github.com/donnemartin/system-design-primer#task-queues)
- Delivery guarantees: at-most-once, at-least-once, exactly-once
  - [karanpratapsingh: Message brokers](https://github.com/karanpratapsingh/system-design#message-brokers)
- Back pressure (what happens when producers outpace consumers)
  - [system-design-primer: Back pressure](https://github.com/donnemartin/system-design-primer#back-pressure)
- Dead letter queues (where failed jobs go and why)
  - [karanpratapsingh: Message queues](https://github.com/karanpratapsingh/system-design#message-queues)
  - [Sidekiq wiki: Error handling](https://github.com/sidekiq/sidekiq/wiki/Error-Handling)
- Idempotency (why "run it again" must be safe)
  - [ByteByteGo: Idempotency](https://github.com/ByteByteGoHq/system-design-101#how-do-we-design-effective-and-safe-apis)

### Project: Payments and Webhooks
[PROJECT.md → Phase 4](PROJECT.md#phase-4--payments-and-webhooks)

### You're done when you can:
- [ ] Explain what happens to an in-flight Sidekiq job when its process is killed
- [ ] Explain why idempotency matters for at-least-once delivery
- [ ] Explain what back pressure is and how Sidekiq handles (or doesn't handle) it
- [ ] Explain the difference between Sidekiq's Redis-backed queue and a
      dedicated message broker like RabbitMQ or Kafka

---

## Phase 5 — Scaling: Load Balancing and Horizontal Scaling

**Why this is fifth:** you can't reason about scaling until you understand what
a single instance does well (Phases 1–4). Now we break out of one box.

### Read
- Vertical vs horizontal scaling (and why vertical has a ceiling)
  - [system-design-primer: Scalability](https://github.com/donnemartin/system-design-primer#performance-vs-scalability)
  - [karanpratapsingh: Scalability](https://github.com/karanpratapsingh/system-design#scalability)
- Load balancer: layer 4 vs layer 7, algorithms (round robin, least connections, IP hash)
  - [system-design-primer: Load balancer](https://github.com/donnemartin/system-design-primer#load-balancer)
  - [karanpratapsingh: Load balancing](https://github.com/karanpratapsingh/system-design#load-balancing)
  - [ByteByteGo: Load balancing](https://github.com/ByteByteGoHq/system-design-101#load-balancing)
- Sticky sessions (why they exist, why they're a trap)
  - covered inside the load balancer sections above
- Stateless vs stateful services (why stateless scales and stateful doesn't)
  - [system-design-primer: Application layer](https://github.com/donnemartin/system-design-primer#application-layer)
- Reverse proxy vs load balancer (they're not the same thing)
  - [system-design-primer: Reverse proxy](https://github.com/donnemartin/system-design-primer#reverse-proxy-web-server)
  - [karanpratapsingh: Reverse proxy](https://github.com/karanpratapsingh/system-design#reverse-proxy)
- Health checks and graceful shutdown
  - [Puma: Restart docs](https://github.com/puma/puma/blob/master/docs/restart.md)

### Project: On-Sale Spike
[PROJECT.md → Phase 5](PROJECT.md#phase-5--on-sale-spike)

### You're done when you can:
- [ ] Explain why session or cache state can't live in process memory with multiple instances
- [ ] Explain what happens to a request that's mid-processing when an instance dies
- [ ] Explain the difference between a load balancer and a reverse proxy
- [ ] Trace a request through the Compose setup: client → Nginx → one of three
      Puma instances → Postgres and back

---

## Phase 6 — Database Scaling

**Why this is sixth:** this is where most system design interviews live, and it
only makes sense after a single database (Phase 2) and multiple app instances (Phase 5).

### Read
- Replication: leader-follower (read replicas), leader-leader
  - [system-design-primer: Replication](https://github.com/donnemartin/system-design-primer#replication)
  - [karanpratapsingh: Database replication](https://github.com/karanpratapsingh/system-design#database-replication)
- Replication lag and its consequences (stale reads, read-your-writes consistency)
  - [system-design-primer: Master-slave replication disadvantages](https://github.com/donnemartin/system-design-primer#disadvantage-replication)
- Rails multiple databases and automatic role switching
  - [Rails guides: Multiple databases](https://guides.rubyonrails.org/active_record_multiple_databases.html)
- Sharding: hash-based vs range-based, choosing a shard key
  - [system-design-primer: Sharding](https://github.com/donnemartin/system-design-primer#sharding)
  - [karanpratapsingh: Sharding](https://github.com/karanpratapsingh/system-design#sharding)
  - [ByteByteGo: Database](https://github.com/ByteByteGoHq/system-design-101#database)
- What queries break when you shard (cross-shard joins, aggregations)
  - covered inside the sharding sections above
- Connection pooling (PgBouncer, why it matters at scale)
  - [PgBouncer docs](https://www.pgbouncer.org/)
- CAP theorem revisited: now with real examples, not just definitions
  - [system-design-primer: CAP theorem](https://github.com/donnemartin/system-design-primer#cap-theorem)
  - [karanpratapsingh: CAP theorem](https://github.com/karanpratapsingh/system-design#cap-theorem)

### Project: Read Replica
[PROJECT.md → Phase 6](PROJECT.md#phase-6--read-replica)

### You're done when you can:
- [ ] Explain read-your-writes consistency using "My tickets" right after checkout
- [ ] Propose a shard key for the app's multi-organizer data and name the first query that breaks
- [ ] Explain why connection pooling matters more as you add replicas and instances
- [ ] Explain CAP using the replica setup: during a partition, do you keep selling
      seats (risk overselling) or stop selling (lose sales)?

---

## Phase 7 — API Design and Service Communication

**Why this is seventh:** designing boundaries between services requires
understanding everything above. This phase is about the edges, not the internals.

### Read
- REST: resource naming, idempotency, status codes, versioning
  - [karanpratapsingh: REST](https://github.com/karanpratapsingh/system-design#representational-state-transfer-rest)
  - [ByteByteGo: API architecture](https://github.com/ByteByteGoHq/system-design-101#api-architecture)
- Pagination: offset vs cursor-based (and why offset breaks at scale)
  - [karanpratapsingh: Pagination](https://github.com/karanpratapsingh/system-design#pagination)
- Rate limiting: token bucket, sliding window
  - [karanpratapsingh: Rate limiting](https://github.com/karanpratapsingh/system-design#rate-limiting)
- API gateway pattern
  - [karanpratapsingh: API gateway](https://github.com/karanpratapsingh/system-design#api-gateway)
  - [ByteByteGo: API gateway](https://github.com/ByteByteGoHq/system-design-101#api-gateway)
- Synchronous (HTTP) vs asynchronous (queue) service communication
  - [system-design-primer: Communication](https://github.com/donnemartin/system-design-primer#communication)
- Circuit breaker pattern (what to do when a downstream service is down)
  - [karanpratapsingh: Circuit breaker](https://github.com/karanpratapsingh/system-design#circuit-breaker)

### Project: Edges and Boundaries
[PROJECT.md → Phase 7](PROJECT.md#phase-7--edges-and-boundaries)

### You're done when you can:
- [ ] Explain why cursor-based pagination scales and offset doesn't, with your own benchmark numbers
- [ ] Explain the circuit breaker pattern using the payment provider going down
- [ ] Explain token bucket rate limiting and why the state lives in Redis

---

## Phase 8 — Observability and Reliability

**Why this is eighth:** you can't improve what you can't see. This phase is about
making invisible problems visible before users find them.

### Read
- The three pillars: logs, metrics, traces
  - [ByteByteGo: Observability](https://github.com/ByteByteGoHq/system-design-101#observability)
- Structured logging (why grep-friendly logs matter)
  - [karanpratapsingh: Logging](https://github.com/karanpratapsingh/system-design#logging)
- Distributed tracing (how to follow a request across services)
  - [karanpratapsingh: Distributed tracing](https://github.com/karanpratapsingh/system-design#distributed-tracing)
- SLIs, SLOs, SLAs (what they actually mean, not just the acronyms)
  - [karanpratapsingh: SLA, SLO, SLI](https://github.com/karanpratapsingh/system-design#sla-slo-sli)
- Alerting: what to alert on, alert fatigue, paging vs notification
  - [karanpratapsingh: Monitoring](https://github.com/karanpratapsingh/system-design#monitoring)
- Incident response: severity levels, postmortems, blameless culture
  - [karanpratapsingh: Incident management](https://github.com/karanpratapsingh/system-design#incident-management)

### Project: Observability
[PROJECT.md → Phase 8](PROJECT.md#phase-8--observability)

### You're done when you can:
- [ ] Explain how tracing would surface a payment that was charged but never
      turned into tickets — a silent failure no error page shows
- [ ] Define an SLO for checkout and explain how you'd measure it
- [ ] Explain why "the server is down" is a bad alert and what a good one looks like
- [ ] Write a short blameless postmortem for one failure you caused on purpose in an earlier phase

---

## Phase 9 — System Design Practice (Full Problems)

**Why this is last:** you now have every building block. This phase is about
combining them under ambiguity, which is what interviews actually test.

### Read
- The 4-step framework: requirements clarification, high-level design,
  deep dive, trade-offs
  - [system-design-primer: How to approach](https://github.com/donnemartin/system-design-primer#how-to-approach-a-system-design-interview-question)
- Practice problems with solutions
  - [system-design-primer: practice problems](https://github.com/donnemartin/system-design-primer#system-design-interview-questions-with-solutions)

### Practice Problems (attempt before reading the solution)
- [ ] Design a URL shortener
- [ ] Design a notification system
- [ ] Design a file sync service (like Dropbox)
- [ ] Design a rate limiter at scale (you built one in Phase 7; now make it global)
- [ ] Design a task queue (you've used Sidekiq; now design one from scratch)
- [ ] **Capstone:** design the ticketing app for 1M concurrent users at on-sale
      (virtual waiting room, inventory sharding, payment at scale)

### For each problem, write in `docs/phases/09-full-problems.md`:
1. Requirements (functional and non-functional)
2. Back-of-envelope estimation (storage, bandwidth, QPS)
3. High-level architecture diagram
4. Data model
5. The single hardest trade-off and your reasoning

---

## Depth Layer (ongoing, no deadline)

- [ ] *Designing Data-Intensive Applications* (Kleppmann), one chapter at a time
- [ ] One engineering blog postmortem per week from
      [kilimchoi/engineering-blogs](https://github.com/kilimchoi/engineering-blogs)

---

## Progress Tracker

| Phase | Topic | Theory | Project | Gate | Tag |
|-------|-------|:------:|:-------:|:----:|:---:|
| 0 | Skeleton | — | ☐ | ☐ | `phase-0` |
| 1 | Request Flow | ☐ | ☐ | ☐ | `phase-1` |
| 2 | Data & Concurrency | ☐ | ☐ | ☐ | `phase-2` |
| 3 | Caching | ☐ | ☐ | ☐ | `phase-3` |
| 4 | Async / Queues | ☐ | ☐ | ☐ | `phase-4` |
| 5 | Load Balancing | ☐ | ☐ | ☐ | `phase-5` |
| 6 | DB Scaling | ☐ | ☐ | ☐ | `phase-6` |
| 7 | API Design | ☐ | ☐ | ☐ | `phase-7` |
| 8 | Observability | ☐ | ☐ | ☐ | `phase-8` |
| 9 | Full Problems | — | ☐ | ☐ | `phase-9` |

**A phase is done when every column is ticked and the tag exists. Not most of them.**
