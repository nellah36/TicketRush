# TicketRush 🎟️

A high-concurrency event ticketing REST API built with Node.js, TypeScript, PostgreSQL, and Redis.

TicketRush is a modular-monolith backend designed to handle thousands of users competing for limited event seats while guaranteeing **zero overselling**, even under concurrent requests.

## 🚀 Key Features

- **Concurrency-Safe Seat Booking:** Prevents multiple users from purchasing the same seat.
- **Atomic Seat Holds:** Uses Redis to temporarily claim seats with expiration.
- **Database-Level Guarantees:** Uses PostgreSQL transactions, row-level locking, and unique constraints.
- **Secure Authentication:** JWT-based authentication with refresh token rotation.
- **Webhook Security:** HMAC signature verification and replay protection.
- **Background Jobs:** BullMQ for asynchronous task processing.
- **Load Testing:** Uses k6 to test system behaviour under concurrent traffic.

## 🛠️ Tech Stack

| Category | Technologies |
|---|---|
| Runtime | Node.js, TypeScript |
| Backend | Express.js |
| Database | PostgreSQL 16, Prisma |
| Caching | Redis 7, ioredis |
| Background Jobs | BullMQ |
| Authentication | JWT, Argon2 |
| Validation | Zod |
| Logging | Pino |
| Testing | Vitest, Supertest |
| Load Testing | k6 |
| Infrastructure | Docker Compose |
| CI/CD | GitHub Actions |

## 🏗️ Architecture

TicketRush follows a modular-monolith architecture with clearly separated business modules.

```text
src/
├── modules/
│   ├── auth/
│   ├── users/
│   ├── events/
│   ├── holds/
│   ├── checkout/
│   ├── webhooks/
│   └── tickets/
│
├── common/
├── config/
└── server.ts
```

Each module follows a layered structure:

**Controller → Service → Repository**

## 🔒 Preventing Overselling

The central engineering challenge is ensuring that no seat can be sold more than once.

TicketRush uses multiple layers of protection:

1. **Redis:** Atomic seat claims using `SET NX EX` to prevent simultaneous holds.
2. **PostgreSQL:** Row-level locking using `SELECT ... FOR UPDATE` to serialize competing transactions.
3. **Unique Constraints:** `UNIQUE(Ticket.seatId)` provides a final database-level guarantee against duplicate ticket sales.

PostgreSQL remains the permanent source of truth. Redis is only used for temporary seat claims.

## ⚙️ Getting Started

### Prerequisites

- Node.js (LTS)
- Docker and Docker Compose
- PostgreSQL 16
- Redis 7

### Installation

```bash
# Clone the repository
git clone <your-repository-url>

# Navigate into the project
cd ticketrush

# Install dependencies
npm install

# Configure environment variables
cp .env.example .env

# Start PostgreSQL and Redis
docker compose up -d

# Run database migrations
npx prisma migrate dev

# Seed the database
npx prisma db seed

# Start development server
npm run dev
```

## 🧪 Testing

```bash
# Run tests
npm test

# Type checking
npx tsc --noEmit

# Run overselling demonstration
npx tsx scripts/oversell-demo.ts

# Run k6 load test
k6 run scripts/k6/loadtest.js
```

## 📚 Project Documentation

- `CLAUDE.md` — AI-assisted development instructions and project conventions.
- `docs/PLAN.md` — Development roadmap and current progress.
- `docs/learnings.md` — Engineering concepts and daily learning notes.

## 🎯 Project Goals

This project is being built as a hands-on exploration of:

- Database concurrency and transaction isolation.
- Distributed locking and race conditions.
- Idempotency and webhook reliability.
- Authentication and refresh token security.
- Backend architecture and scalability.
- Performance testing under concurrent workloads.

---

**Status:** Under Development
