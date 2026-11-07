# TicketRush — Learning + Build Plan

**Budget:** 6 focused hours/day · **Target:** ~20 days · **Rule:** no phase starts until the previous gate is green.

Every phase runs the same loop:
**Stack → Learn → Build → Break → Understand → Gate ✅ → `git tag phase-N`**

Legend:
- **L1 Use** (with docs) · **L2 Build** (without help) · **L3 Defend** (internals, trade-offs, failure modes)
- 🧑 = YOU write it (critical path) · 🤖 = Claude may scaffold it · ⭐ = interview centerpiece

---

## Daily 6-hour template

| Block | Time | Do |
|---|---|---|
| 1. Recall | 15 min | No notes: write what you built yesterday and why. Read yesterday's "next step". |
| 2. Learn | 75 min | `/teach <concept>` → primary source → quiz. Meanwhile 🤖 scaffolds today's boilerplate. |
| 3. Predict | 15 min | Sketch today's design on paper + how it could fail. |
| 4. Build | 2.5 h | Plan mode → compare to your sketch → 🧑 write critical code. 10-min break halfway. |
| 5. Break + test | 60 min | Run the break-it drill. Make gate tests pass. |
| 6. Lock in | 30 min | Explain aloud, `learnings.md` entry, 5 flashcards, commit, write tomorrow's first step. |

Stuck rule: **30 min alone → `/hint 1` → 15 min → `/hint 2` → 15 min → `/hint 3`.**
Every 7th day: lighter review day — `/grill`, flashcards, redraw the 4 diagrams from memory.

## Progress tracker

- [ ] Day 0 — Setup
- [ ] Phase 0 — Foundation (Days 1–3)
- [ ] Phase 1 — Auth & access (Days 4–6)
- [ ] Phase 2 — Redis layer (Day 7)
- [ ] Phase 3 ⭐ — Holds / race condition (Days 8–10)
- [ ] Phase 4 ⭐ — Checkout transaction (Days 11–13)
- [ ] Phase 5 — Payment webhook & tickets (Days 14–15)
- [ ] Phase 6 — Workers (Day 16)
- [ ] Phase 7 — Proof & polish (Days 17–19)
- [ ] Interview prep (Days 20–21)

Weekly checkpoints: **end of week 1** = Phases 0–2 · **end of week 2** = Phases 3–4 · **end of week 3** = shipped.
Behind by 2+ days? Cut polish, never the proofs.

---

## Day 0 — Setup (half day)

**Goal:** tools ready; never touch tooling again.

- [ ] Install: Node LTS, Docker Desktop, Git, k6, Claude Code (VS Code extension or CLI)
- [ ] Put this starter kit in the repo root: `CLAUDE.md`, `.claude/commands/`, `docs/`
- [ ] Verify the commands work in Claude Code: `/teach`, `/hint`, `/grill`, `/explain-back`, `/phase-start`, `/phase-done`
- [ ] Create `.gitignore` (node_modules, .env, dist), commit, push
- [ ] Optional: connect the GitHub connector

**Gate ✅** `docker --version`, `node -v`, `k6 version` all work · repo pushed with starter kit.

---

## Phase 0 — Foundation · Days 1–3

**Goal:** one-command boot, talks to Postgres + Redis, event CRUD with tests.

### Learn
| Concept | Level | Done when you can… | Source |
|---|---|---|---|
| TypeScript strict: types, unions, narrowing, generics basics, `unknown` vs `any` | L2 | Type an Express handler and a Prisma result without `any` | TS Handbook: Everyday Types, Narrowing |
| Express middleware order + error middleware | L2 | Explain why the error handler has 4 args and must be last | Express guide: Using middleware, Error handling |
| Docker + Compose: images, containers, volumes, healthchecks | L2 | Explain why Postgres needs a volume and Redis doesn't | Docker docs: Compose quickstart |
| Relational design: PK/FK, 1-N vs N-N, UNIQUE, CHECK | L2–L3 | Draw the full ERD from memory | Postgres docs: Constraints |
| Prisma: schema, relations, migrate dev vs deploy, seed | L2 | Add a model, migrate, seed without looking anything up | Prisma docs: Schema, Migrate |
| Zod: schemas, `safeParse`, `z.infer`, env validation | L2 | Make the app refuse to start on a missing env var | Zod docs |
| Layered architecture: controller → service → repository | L3 | Say what each layer may and may not know | — |

### Build
- [ ] 🤖 `docker-compose.yml`: postgres (volume, healthcheck), redis (healthcheck)
- [ ] 🤖 TS project: `tsconfig` (strict), `package.json` scripts, ESLint/Prettier, Vitest
- [ ] 🤖 `src/config/env.ts` (Zod), `prisma.ts`, `redis.ts` singletons
- [ ] 🤖 `src/common/`: logger (Pino + request ID), AppError classes, error + 404 middleware, `validate()` middleware
- [ ] 🤖 `app.ts` / `server.ts` with graceful shutdown; `GET /health` (checks DB + Redis)
- [ ] 🧑 **Design the FULL schema on paper first** (all 11 tables, constraints, enums)
- [ ] 🧑 `schema.prisma`: migrate `User`, `Venue`, `Event`, `TicketTier`, `Seat` (with UNIQUE + CHECK)
- [ ] 🤖 `seed.ts`: 1 organizer, 1 venue, 1 event, 2 tiers, 100 seats
- [ ] 🧑 Events module (with 🤖 help on boilerplate): `GET /events`, `GET /events/:id`, `POST /events`, `PATCH /events/:id`, `POST /events/:id/publish`, `POST /events/:id/cancel`
- [ ] 🧑 Tiers & seats: `POST /events/:id/tiers`, `POST /events/:id/seats/bulk`
- [ ] 🧑 Event state machine: DRAFT → PUBLISHED → CANCELLED (reject illegal moves)
- [ ] 🤖 Integration test setup (test DB, reset between tests) + 🧑 the test cases

### Break it
- Send malformed JSON / wrong types → see exactly which layer rejects it.
- Insert a duplicate seat → see the unique constraint error → map it to a 409.
- Stop Postgres → see what `/health` and the error handler do.

### Understand (answer without notes)
1. Why a modular monolith instead of microservices?
2. Why separate service from repository?
3. Why `UNIQUE(eventId, section, row, number)` — what bug does it prevent?
4. What's the difference between `prisma migrate dev` and `migrate deploy`?
5. Why validate env at startup instead of at first use?

### Gate ✅
- [ ] Fresh clone → `docker compose up -d` + migrate + seed → app runs
- [ ] `/health` reports DB and Redis OK
- [ ] Event CRUD integration tests pass; illegal state transitions rejected
- [ ] Invalid input → 400 with `{ error: { code, message } }`
- [ ] `npx tsc --noEmit` clean

---

## Phase 1 — Auth & access · Days 4–6

**Goal:** secure identity with rotation, reuse detection, roles and ownership.

### Learn
| Concept | Level | Done when you can… | Source |
|---|---|---|---|
| Hashing vs encryption, salts, argon2id vs bcrypt vs SHA-256 | L3 | Explain why "slow" is a feature | OWASP Password Storage Cheat Sheet |
| JWT: header/payload/signature, stateless, can't be revoked | L3 | Decode a JWT by hand and explain each part | jwt.io intro, OWASP JWT Cheat Sheet |
| Access/refresh split, rotation, token families, reuse detection | L3 | Walk through a token-theft attack and how reuse detection catches it | Auth0: Refresh Token Rotation |
| Cookies: httpOnly, Secure, SameSite (Strict/Lax/None) | L3 | Say which flag stops which attack | MDN: Set-Cookie |
| XSS vs CSRF, CORS preflight | L2 | Explain why CORS is not server security | OWASP CSRF Cheat Sheet, MDN CORS |
| RBAC vs ownership; 401 vs 403 | L2 | Give one example of each status from your API | — |

### Build
- [ ] 🧑 Migrate `RefreshToken` (tokenHash, familyId, expiresAt, revokedAt) + `role` enum on User
- [ ] 🧑 `POST /auth/register`, `POST /auth/login` (argon2)
- [ ] 🧑 ⭐ `POST /auth/refresh`: rotation + **reuse detection → revoke family**
- [ ] 🧑 `POST /auth/logout` (revoke + clear cookie), `GET /me`
- [ ] 🧑 `authenticate` middleware (verify access JWT → `req.user`)
- [ ] 🧑 `authorize(...roles)` middleware + ownership check on event writes
- [ ] 🤖 helmet, cors, cookie-parser wiring; auth Zod schemas; test helpers (login as role X)
- [ ] 🧑 Auth integration tests

### Break it
- Replay an old refresh token after rotating → whole family revoked.
- Use an expired access token → 401. Tamper one character of a JWT → 401.
- Organizer A edits organizer B's event → 403. Attendee creates event → 403.

### Understand
1. Why store only the hash of the refresh token? What if the DB leaks?
2. Why 15 minutes for the access token? What's the trade-off?
3. Why refresh token in an httpOnly cookie but access token in memory?
4. Why not server sessions?
5. What if two tabs refresh at the same time? (Know the problem; it's a great discussion point.)

### Gate ✅
- [ ] Refresh-token replay revokes the family (test)
- [ ] RBAC + ownership tests pass (401 vs 403 correct)
- [ ] Refresh cookie: `httpOnly`, `Secure` (in prod), `SameSite` set
- [ ] No raw refresh token ever stored (check the DB)

---

## Phase 2 — Redis layer · Day 7

**Goal:** fast reads, abuse protection, graceful degradation.

### Learn
| Concept | Level | Done when you can… | Source |
|---|---|---|---|
| Redis data model, `GET/SET/DEL/INCR/EXPIRE/TTL`, `redis-cli` | L2 | Inspect keys and TTLs live | Redis docs: Strings |
| Cache-aside, invalidation, TTL, stale reads, thundering herd | L3 | Explain when stale data is OK in TicketRush and when it isn't | AWS caching best practices |
| Rate limiting: fixed vs sliding window vs token bucket | L2 | Explain the window-edge burst problem | Redis docs: rate limiting patterns |
| Fail-open vs fail-closed | L3 | Defend your choice for cache vs rate limiter | — |

### Build
- [ ] 🧑 Cache-aside on `GET /events/:id` (5 min) and `GET /events` list (60 s)
- [ ] 🧑 Invalidate on update / publish / cancel
- [ ] 🧑 Rate limiter middleware `rl:{route}:{ip}` with `INCR` + `EXPIRE` → 429 + `Retry-After`
- [ ] 🧑 If Redis is down: cache is skipped (reads still work)
- [ ] 🤖 Tests for hit/miss/invalidation and 429

### Break it
- Update an event without invalidating → see stale data → fix.
- `docker compose stop redis` → event reads still work from Postgres.

### Understand
1. Why cache-aside and not write-through?
2. Why is the list TTL shorter than the detail TTL?
3. What is a thundering herd and how would you mitigate it?
4. Should the rate limiter fail open or closed when Redis dies? Why?

### Gate ✅
- [ ] Test proves second read is a cache hit, and update invalidates it
- [ ] Request N+1 in the window → 429
- [ ] Redis stopped → event reads still succeed

---

## Phase 3 ⭐ — Holds: the race condition · Days 8–10

**Goal:** reproduce the oversell, prove it, then make it impossible at the hold layer.

### Learn
| Concept | Level | Done when you can… | Source |
|---|---|---|---|
| Node event loop: how `await` lets other requests interleave | L3 | Answer "Node is single-threaded, how did it race?" in 30 s | Node docs: Event loop |
| Race conditions, check-then-act, critical sections | L3 | Draw a 2-request timeline that oversells | DDIA ch. 7 (race conditions) |
| Atomic operations, compare-and-set | L3 | Explain why `GET` then `SET` isn't atomic but `SET NX` is | Redis docs: SET |
| Redis single-threaded command execution | L3 | Explain why two `SET NX` can't both win | Redis docs |
| `SET NX EX` vs `SET NX` + `EXPIRE` | L3 | Describe the crash that makes the second one a bug | Redis docs: SET options |
| Distributed lock caveats (TTL expiry mid-work, Redlock debate) | L2 | Say why a hold isn't a perfect lock and why Postgres backs it | Redis docs: distributed locks |

### Build — in this order
- [ ] 🧑 Migrate `Reservation` + `ReservationSeat`
- [ ] 🧑 **Naive hold** (on branch `naive-holds`): `GET hold:…` → if empty → `SET`
- [ ] 🤖 skeleton / 🧑 logic: `scripts/oversell-demo.ts` — 50 concurrent holds on 1 seat, print winner count
- [ ] 🧑 Run it. **Record the result + screenshot** → `docs/evidence/`
- [ ] 🧑 ⭐ Fix: `SET hold:{eventId}:{seatId} userId NX EX 600`
- [ ] 🧑 ⭐ Multi-seat holds: on any failure, release seats already taken → 409
- [ ] 🧑 Check seats not SOLD + event PUBLISHED before holding
- [ ] 🧑 Create Reservation (HELD, expiresAt) + ReservationSeats; `DELETE /holds/:id` (release)
- [ ] 🧑 `GET /events/:id/seats`: DB status + Redis `MGET` → AVAILABLE | HELD | SOLD
- [ ] 🤖 Integration tests scaffolding → 🧑 concurrency test cases

### Break it
- Naive version: run the demo 5 times, record winners each time (expect > 1).
- Fixed version: 10 runs → exactly 1 winner every time.
- Multi-seat: user A holds [s1,s2], user B requests [s2,s3] → B gets 409 and **s3 is not left held**.

### Understand
1. Draw the naive oversell timeline.
2. "Node is single-threaded — how did it race?" Also: what changes with 3 API instances?
3. Why must `EX` be in the same command as `NX`?
4. What if Redis is wiped right after a hold succeeds?
5. Why is seat status in Postgres only AVAILABLE/SOLD, not HELD?

### Gate ✅
- [ ] Naive demo shows > 1 winner (evidence saved, branch kept)
- [ ] Fixed demo: exactly 1 winner across 10 runs
- [ ] Partial multi-seat failure leaves no orphaned holds (test)
- [ ] Seat map correctly shows HELD seats

---

## Phase 4 ⭐ — Checkout transaction · Days 11–13

**Goal:** correctness guaranteed by Postgres even if Redis lies; no duplicate orders.

### Learn
| Concept | Level | Done when you can… | Source |
|---|---|---|---|
| ACID — mapped to *your* checkout | L3 | Point at your code for each letter | DDIA ch. 7 |
| Isolation levels + anomalies (dirty read, non-repeatable, phantom, lost update) | L3 | Explain why Read Committed + `FOR UPDATE` beats SERIALIZABLE here | Postgres docs: Transaction Isolation |
| `SELECT … FOR UPDATE`, `NOWAIT`, `SKIP LOCKED` | L3 | Say what blocks what, and when you'd use the other two | Postgres docs: Explicit Locking |
| Pessimistic vs optimistic locking (version column) | L3 | Compare with your Peachy Shop choice | DDIA ch. 7 |
| Deadlocks: Coffman conditions, lock ordering, Postgres detection | L3 | Reproduce one and explain why sorting fixes it | Postgres docs: Deadlocks |
| Idempotency keys | L3 | Handle same key + different body correctly | Stripe: Idempotent requests |
| Prisma interactive `$transaction` + `$queryRaw` | L2 | Explain why `FOR UPDATE` needs raw SQL in Prisma | Prisma docs: Transactions |
| State machine pattern | L2 | Unit-test illegal transitions | — |

### Build
- [ ] 🧑 Migrate `Order` (status enum, total, idempotencyKey; unique on `userId + idempotencyKey`)
- [ ] 🧑 Order state machine class: PENDING → PAID → REFUNDED, PENDING → FAILED (+ unit tests)
- [ ] 🧑 ⭐ `POST /checkout` with `Idempotency-Key` header:
  - existing order for (user, key) → return it unchanged (different body → 422)
  - verify all Redis holds belong to this user → else 409
  - **transaction:** `SELECT … FOR UPDATE` on **sorted** seat IDs → all AVAILABLE? reservation HELD & not expired? → create PENDING order → commit
  - return mock payment intent `{ orderId, amount, intentId }`
- [ ] 🧑 `GET /orders`, `GET /orders/:id` (ownership)
- [ ] 🤖 skeleton / 🧑 logic: `scripts/deadlock-demo.ts`

### Break it
- **Deadlock drill:** two transactions lock [s1,s2] and [s2,s1] unsorted → Postgres aborts one. Sorted → no deadlock.
- Send the same checkout twice concurrently → one order.
- Delete the Redis holds manually, mark a seat SOLD, try checkout → Postgres refuses.

### Understand
1. Why lock in Postgres if Redis already holds the seat?
2. Why Read Committed + `FOR UPDATE` instead of SERIALIZABLE?
3. Why sort the IDs? Draw the deadlock.
4. Same Idempotency-Key, different body — what do you return and why?
5. Why pessimistic locking here when you used optimistic in Peachy Shop?
6. What happens if the API crashes mid-transaction?

### Gate ✅
- [ ] Deadlock demo: unsorted deadlocks, sorted doesn't (both recorded in `docs/evidence/`)
- [ ] Duplicate checkout (same key) → exactly one order (test)
- [ ] Redis holds wiped + seat SOLD → checkout fails (test)
- [ ] Illegal state transitions rejected (unit tests)

---

## Phase 5 — Payment webhook & tickets · Days 14–15

**Goal:** verified, replay-safe payment confirmation that issues each ticket once.

### Learn
| Concept | Level | Done when you can… | Source |
|---|---|---|---|
| HMAC vs plain hash vs signature | L3 | Explain why a shared secret proves origin | Stripe: Webhook signatures |
| Raw-body signing | L3 | Explain why parsed JSON breaks the signature | Express docs: `express.raw` |
| Constant-time comparison (`timingSafeEqual`) | L3 | Explain the timing attack | Node docs: crypto |
| Replay protection with timestamps | L2 | Defend the 5-minute window | Stripe docs |
| At-least-once delivery | L3 | Explain why handlers must be idempotent | DDIA ch. 11 |

### Build
- [ ] 🧑 Migrate `Ticket` (**UNIQUE seatId**, qrHash) + `AuditLog`
- [ ] 🤖 `scripts/mock-payment.ts`: signs and sends a webhook (success/failure)
- [ ] 🧑 ⭐ `POST /webhooks/payment` with `express.raw()`: verify `HMAC-SHA256(secret, timestamp + "." + rawBody)` via `timingSafeEqual`; reject old timestamps
- [ ] 🧑 ⭐ One transaction: lock order → already PAID? return 200 no-op → lock seats (sorted) → all AVAILABLE? (else order FAILED) → order PAID → seats SOLD → tickets → reservation CONFIRMED
- [ ] 🧑 After commit: `DEL` Redis holds, write AuditLog
- [ ] 🧑 `GET /tickets` (mine)

### Break it
- Tamper one byte of the body → 401. Old timestamp → 401.
- Send the same webhook 5 times → tickets issued once, every response 200.

### Understand
1. Why sign the raw body?
2. Why `timingSafeEqual` instead of `===`?
3. Why can't Redis cleanup be inside the DB transaction? What if it fails after commit?
4. Why return 200 on a replay?
5. What if the webhook arrives before the checkout commits?

### Gate ✅
- [ ] Tampered signature / body → 401 (test)
- [ ] 5× same webhook → one set of tickets (test)
- [ ] Audit log entry per paid order

---

## Phase 6 — Workers · Day 16

**Goal:** the system heals itself.

### Learn
| Concept | Level | Done when you can… | Source |
|---|---|---|---|
| Queues: producer, consumer, retry, backoff, dead-letter | L2 | Explain each in one line | BullMQ docs: Guide |
| BullMQ repeatable jobs, separate worker process | L2 | Explain why the worker isn't inside the API | BullMQ docs |
| Eventual consistency, reconciliation | L3 | Explain the "two clocks" problem (Redis TTL vs `expiresAt`) | DDIA ch. 5/9 |
| Idempotent jobs | L3 | Explain why `WHERE status = 'HELD'` makes it safe to rerun | — |

### Build
- [ ] 🤖 `src/jobs/queues.ts`, `worker.ts` entry, `worker` service in Compose
- [ ] 🧑 Expire-holds job (every 1 min): HELD + past `expiresAt` → EXPIRED; DEL leftover keys
- [ ] 🧑 Reconcile job (every 5 min): `SCAN hold:*` → no matching HELD reservation → log + DEL
- [ ] 🧑 Tests with a short TTL

### Break it
- Kill the worker mid-run, restart → recovers.
- Run the expiry job twice → no extra effect.

### Understand
1. What breaks if the worker is down for an hour?
2. Why both a Redis TTL and a Postgres `expiresAt`?
3. Why `SCAN` and not `KEYS`?

### Gate ✅
- [ ] Expired hold → seat becomes available (test)
- [ ] Job run twice = no change (test)
- [ ] Worker restart recovers cleanly

---

## Phase 7 — Proof & polish · Days 17–19

**Goal:** evidence + a README that tells the story.

### Learn
| Concept | Level | Done when you can… | Source |
|---|---|---|---|
| k6: VUs, scenarios, checks, thresholds | L2 | Write a scenario without looking | k6 docs |
| Latency percentiles p50/p95/p99, throughput | L2 | Explain why p99 matters more than average | k6 docs: metrics |
| GitHub Actions with service containers | L1 | Read your workflow and explain each step | GitHub docs |
| Scaling talk: replicas, read replicas, waiting room, sharding by event | L2 | Answer "what at 100× scale?" in 2 min | DDIA ch. 1, 6 |

### Build
- [ ] 🤖 `.github/workflows/ci.yml` (Postgres + Redis services, typecheck, tests)
- [ ] 🧑 `scripts/k6/loadtest.js`: 1,000 users × 100 seats; verify via SQL afterwards
- [ ] 🧑 ⭐ Redis-wipe test: `FLUSHALL` mid-run → 0 oversold
- [ ] 🤖 README skeleton → 🧑 fill: problem, architecture diagram, 3 walls of defense, ADRs, numbers, future work
- [ ] 🧑 `docs/adr/` — at least 5 decisions with rejected alternatives
- [ ] 🧑 Resume bullets with your real numbers

### Gate ✅ = Definition of Done
- [ ] `docker compose up` + two commands runs everything
- [ ] Oversell script: exactly 1 winner
- [ ] k6: 100 sold, 0 oversold; p95 recorded
- [ ] Redis wiped mid-test: 0 oversold
- [ ] Refresh replay revokes family · duplicate checkout = 1 order · webhook replay = no double tickets
- [ ] CI green
- [ ] README has diagram, decisions, numbers
- [ ] You can explain every file without looking

---

## Interview prep · Days 20–21

- [ ] Redraw all 4 diagrams from memory (runtime, inside-API, data, flows C/D/E)
- [ ] Rehearse 3 pitches aloud: **30 s** (problem → fix → proof), **2 min** (3 walls + numbers), **10 min** (whiteboard)
- [ ] `/grill` × 3 sessions: concurrency, auth/security, scaling
- [ ] Review all flashcards; redo every one you missed
- [ ] Prepare 3 bug stories from `learnings.md` (what broke → how found → fix)
- [ ] Prepare 5 "why not X?" answers: Lua scripts, SERIALIZABLE, HELD in Postgres, sessions, microservices
- [ ] Be ready to say honestly how you used Claude (tutor + reviewer; you wrote the core)
