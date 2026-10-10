# The Project: a Ticketing App

One app that grows across every phase of [learning-path.md](learning-path.md).
Event ticketing is a domain everyone already understands, and it naturally hits
every concept in the path: contended inventory, payments, traffic spikes at
on-sale, read-heavy browsing, and multi-tenant data.

Each phase below lists what to **build**, what to **break** on purpose, and the
**proof** — specs or scripts that demonstrate the concept actually holds, not just
that the code runs.

---

## Domain

| Model | Notes |
|-------|-------|
| `Organizer` | The tenant. Owns events. Natural shard key candidate in Phase 6 |
| `Venue` | Has a fixed seat layout |
| `Seat` | `venue_id`, section, row, number |
| `Event` | `organizer_id`, `venue_id`, `starts_at`, `on_sale_at` |
| `EventSeat` | Per-event inventory: `event_id`, `seat_id`, `status` (available / held / sold), `price_cents`, `held_until` |
| `User` | Buyer |
| `Order` | `user_id`, `event_id`, `status` (pending / paid / failed / expired), `total_cents` |
| `Ticket` | `order_id`, `event_seat_id` |
| `Payment` | `order_id`, `provider_ref`, `status` — added in Phase 4 |

## Endpoints (grow over time)

| Endpoint | Purpose | Introduced |
|----------|---------|:----------:|
| `GET /events` | List with filters (city, date range) | 2 |
| `GET /events/:id` | Event with seat availability | 1 |
| `POST /events/:id/holds` | Hold N seats for a few minutes | 2 |
| `POST /orders` | Turn a hold into an order | 2 |
| `POST /webhooks/payments` | Payment provider callback | 4 |
| `GET /me/tickets` | The buyer's tickets | 2 |

## Repository layout

```
docker-compose.yml       ← whole system, one command (Phase 0)
ticketing/               ← the Rails app (Phase 0)
payment-provider/        ← tiny fake payment service (Phase 4)
nginx/                   ← load balancer config (Phase 5)
scripts/                 ← load tests, benchmarks, failure drills
docs/phases/NN-*.md      ← notes per phase
```

---

## Phase 0 — Skeleton

**Build**
- `rails new ticketing --api -d postgresql --skip-solid` — Sidekiq and Redis are
  the stack here, so skip the Solid Queue / Solid Cache defaults
- `docker-compose.yml` with `ticketing`, `postgres`, `redis` — exact image versions
- Migrations for the domain above (no `Payment` yet)
- `db/seeds.rb` that is **deterministic** (fixed `Random` seed) and **sized by env**,
  e.g. `SEED_SCALE=small` for dev and `SEED_SCALE=large` (millions of `EventSeat`
  rows) for Phase 2's query work
- `bin/setup` that creates, migrates, and seeds with no prompts

**Proof**
- Fresh clone → `docker compose up -d && docker compose exec ticketing bin/setup` → app responds
- Seeding twice gives the same row counts and the same first/last IDs

## Phase 1 — Request Tracer

**Build**
- Rack middleware that records a timeline for each request: middleware stack,
  routing, controller, each SQL query, serialization
- `GET /events/:id` as the traced endpoint
- `scripts/trace.sh` that uses `curl -w` to capture DNS, TCP connect, TLS (if any),
  time to first byte, and total — then prints it alongside the server timeline

**Break**
- Stop Postgres mid-request; stop Redis; send a malformed request. Note where each one fails

**Proof**
- One printed timeline that names every hop from "enter" to "response"

## Phase 2 — Booking Core + Query Clinic

**Build**
- Holds: `POST /events/:id/holds` marks seats `held` with `held_until`
- Orders: `POST /orders` turns a hold into a paid order (payment is faked as instant success for now)
- `seats_available` counter on `Event` (deliberate denormalization)
- **Query Clinic:** write three queries naively first, against `SEED_SCALE=large`:
  1. Events in a city in a date range, ordered by date
  2. Available seats for an event, grouped by section
  3. Organizer sales report (revenue per event, last 30 days)

  For each: capture EXPLAIN ANALYZE, diagnose, fix, capture again, and write up why
  the fix works in `docs/phases/02-data-concurrency.md`

**Break**
- Fire 50 concurrent purchases at the last seat with no locking. Count the oversells

**Proof**
- Spec: 50 threads buying the last seat → exactly 1 order succeeds
- Spec: `seats_available` stays correct under concurrent purchases
- Before/after query plans committed in the notes

## Phase 3 — Cache Layer

**Build**
- Redis cache for `GET /events` and `GET /events/:id`
- Two strategies for seat availability behind a feature flag: **cache-aside** and
  **write-through**
- Stampede protection for a hot event (lock or early recompute)
- ETags / conditional GET on event pages

**Break**
- Expire a hot event's cache under load. Stop Redis entirely

**Proof**
- `scripts/bench_cache.*` comparing no cache vs both strategies (numbers in notes)
- Spec: buying a seat makes the cached availability correct (per strategy)
- Spec: 100 concurrent cache misses on one key → 1 database query

## Phase 4 — Payments and Webhooks

**Build**
- `payment-provider/` — a tiny Sinatra (or plain Rack) service that accepts a charge
  and later POSTs a webhook back. Configurable to **delay, duplicate, fail, or drop**
  webhooks
- Checkout now creates a pending order and calls the provider; the webhook is
  processed by a Sidekiq job that marks the order paid and issues tickets
- Idempotent webhook handling, retries with exponential backoff, dead-letter
  handling after max retries
- Jobs: confirmation email, expiring stale holds back to `available`
- Queue depth and failure rate visible (Sidekiq Web is fine)

**Break**
- Kill the worker mid-job. Send every webhook twice. Flood checkout faster than
  workers can drain

**Proof**
- Spec: the same webhook delivered twice → one `Payment`, one set of tickets
- Spec: expired holds return to `available`
- Failure drills written up: what happened, what you expected, why they differed

## Phase 5 — On-Sale Spike

**Build**
- Nginx in front of 3 `ticketing` instances, round robin
- Health check endpoint Nginx uses to drop unhealthy instances
- Graceful shutdown: in-flight requests finish before an instance stops
- Audit for in-process state (memory cache store, class-level variables, local
  files) and move it out

**Break**
- Load test an on-sale moment (e.g. `k6`, `oha`, or `hey`). Kill one instance
  mid-test, then two

**Proof**
- `scripts/load_onsale.*` with results in notes: error count with and without graceful shutdown
- A rolling restart under load with zero failed requests

## Phase 6 — Read Replica

**Build**
- Postgres primary + one streaming replica in Compose
- Rails multiple databases: writes to primary, reads to replica, using automatic
  role switching
- Read-your-writes for `GET /me/tickets` right after checkout
- PgBouncer in front of the primary
- On paper (in notes): shard `EventSeat`/`Order` by organizer — what breaks first?
  *Stretch:* actually shard with Rails horizontal sharding

**Break**
- Add artificial replication lag (`recovery_min_apply_delay` on the replica) and
  see which pages go stale

**Proof**
- Spec / script: with 5s replica lag, a buyer sees their tickets immediately after checkout
- Connection counts with and without PgBouncer under the Phase 5 load test

## Phase 7 — Edges and Boundaries

**Build**
- Versioned API (`/v1`), consistent error format, correct status codes
- Cursor pagination on `GET /events`
- Token-bucket rate limiter in Redis as Rack middleware at the edge; a stricter
  limit on holds per user during on-sale
- Circuit breaker around the payment provider client: 3 consecutive 5xx → open
  for 30s → half-open

**Break**
- Set the payment provider to fail 100%, then 30%. Hammer holds from one user

**Proof**
- Benchmark: offset vs cursor at a deep page (numbers in notes)
- Spec: limiter allows a burst, then returns 429 with `Retry-After`
- Spec: breaker opens after 3 failures, rejects fast while open, recovers half-open

## Phase 8 — Observability

**Build**
- Structured JSON logs with a request ID that follows a request through
  `ticketing` → Sidekiq job → `payment-provider` → webhook back
- `/metrics`: request count, latency percentiles, error rate, queue depth
- A dashboard script that polls `/metrics` and prints a summary
- An alert rule: checkout error rate > 5% over 1 minute

**Break**
- Make the provider drop 10% of webhooks silently. Can you find the affected
  orders from logs and metrics alone?

**Proof**
- One request ID grepped across all three services' logs, showing the full path
- The alert fires during the break drill and clears afterward
- A blameless postmortem for one earlier drill

## Phase 9 — Full Problems

No new code. Paper designs in `docs/phases/09-full-problems.md`, ending with the
capstone: this app at 1M concurrent users.

---

## Checkpoints

Every finished phase gets an **annotated git tag**. `phase-0` is the bare
skeleton; `phase-N` is the state when phase N is complete. The start of phase N
is simply `phase-(N-1)`.

### Before tagging a phase

- [ ] Fresh clone → one command → app runs, no manual steps
- [ ] Seeds are deterministic
- [ ] Versions are pinned (`.ruby-version`, `Gemfile.lock`, exact Docker image tags)
- [ ] Every proof spec/script for the phase passes
- [ ] `docs/phases/NN-*.md` is written, including gate answers
- [ ] Progress tracker in `learning-path.md` is updated

```bash
git tag -a phase-3 -m "Caching: cache-aside vs write-through, stampede protection"
git push origin phase-3
gh release create phase-3 --notes-file docs/phases/03-caching.md
```

### Rules

- **Never move a tag.** Found a phase-2 bug while in phase 5? Fix it forward on `main`.
- **`main` is the first run.** Redos happen on branches.

### Re-learning a phase

```bash
git switch -c redo/phase-3 phase-2   # exactly where you stood before phase 3
```

Redo the phase from there, without later code giving away answers. Push the
branch if you like — comparing two attempts months apart is half the value.

### Comparing your work

```bash
git diff phase-2 phase-3 -- ticketing/   # everything phase 3 changed
git diff phase-3 redo/phase-3            # first attempt vs redo
```
