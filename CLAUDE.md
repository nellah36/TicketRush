# TicketRush — Project Memory for Claude

## What this is
A modular-monolith REST backend for high-contention event ticketing.
Organizers publish events; thousands of attendees compete for limited seats.
Goal: ZERO oversells under concurrent load, proven by scripts and a k6 load test.

I am building this to LEARN and to DEFEND it in interviews. Read "Learning mode" below before doing anything.

## Core principle (never violate)
Redis = fast, temporary claim. PostgreSQL = permanent truth.
Every correctness guarantee must hold even if Redis is wiped (FLUSHALL).

Defense in depth against oversells:
1. Redis `SET hold:{eventId}:{seatId} userId NX EX 600` (atomic claim)
2. Postgres `SELECT ... FOR UPDATE` on seats, sorted by id, plus status check
3. `UNIQUE(Ticket.seatId)` as the final backstop

## Stack
Node.js LTS, TypeScript (strict), Express, PostgreSQL 16, Prisma (+ $queryRaw for FOR UPDATE),
Redis 7 via ioredis, BullMQ, Zod, jsonwebtoken, argon2, helmet, cors, cookie-parser,
Pino + pino-http, Vitest + Supertest, k6, Docker Compose, GitHub Actions.

## Architecture rules
- Modules: auth, users, events, holds, checkout, webhooks, tickets (in src/modules/)
- Each module: controller -> service -> repository, plus schemas.ts (Zod)
  - Controller: HTTP only. Service: business rules + transactions. Repository: Prisma only.
- Shared code in src/common (errors, middleware, logger) and src/config (env, prisma, redis singletons)
- Middleware order: requestId -> pino-http -> helmet -> cors -> cookie-parser ->
  express.raw (webhooks only) / express.json -> rateLimit -> authenticate -> authorize -> validate
- Errors: throw AppError subclasses; one error handler maps them to `{ error: { code, message } }`
- ALWAYS sort seat IDs before locking (deadlock prevention)
- Store only HASHES of refresh tokens
- Webhook HMAC is computed over the RAW body, compared with crypto.timingSafeEqual
- Redis cleanup happens AFTER a Postgres commit, never inside the transaction
- Seat status in Postgres is only AVAILABLE | SOLD (no HELD; holds live in Redis + Reservation)

## Out of scope (do NOT add; suggest docs/future-work.md instead)
Lua scripts, waiting room, SSE, microservices, real payments, frontend, Swagger polish, Kubernetes.

## Learning mode (IMPORTANT)
- I write the CRITICAL PATHS myself: hold logic, checkout transaction + FOR UPDATE,
  idempotency keys, webhook HMAC + replay safety, refresh rotation + reuse detection,
  expiry/reconciliation logic. For these: do NOT write the implementation.
  Ask guiding questions, give hints in levels (1 = concept, 2 = pseudocode, 3 = partial code),
  and review my code like a strict senior engineer.
- You MAY write: Docker/config, boilerplate CRUD, error classes, logger, test scaffolding,
  seed data, scripts skeletons, README formatting.
- When I make a design choice, ask what alternative I rejected and why.
- After each feature, ask me 5 interview-style questions, including one "what if X fails?".
  Don't reveal answers until I attempt them.
- Explain WHY, not just WHAT. Prefer primary sources (Postgres docs, Redis docs, DDIA).

## Commands
- Start infra: `docker compose up -d`
- Migrate: `npx prisma migrate dev`
- Seed: `npx prisma db seed`
- Dev server: `npm run dev`
- Tests: `npm test`
- Typecheck: `npx tsc --noEmit`
- Oversell demo: `npx tsx scripts/oversell-demo.ts`
- Load test: `k6 run scripts/k6/loadtest.js`

## Progress
Current phase is tracked in docs/PLAN.md. Daily notes in docs/learnings.md.
