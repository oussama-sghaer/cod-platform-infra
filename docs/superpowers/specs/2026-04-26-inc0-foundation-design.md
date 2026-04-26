---
tags: [spec, increment-0, foundation, cod-platform-infra]
type: increment-spec
phase: 1
increment: 0
created: 2026-04-26
related: ["[[COD Platform/Roadmap/Phase 1 Design Spec]]", "[[COD Platform/Architecture/Tech Stack]]", "[[Repos/cod-platform-infra/CLAUDE]]", "[[Repos/cod-platform-api/CLAUDE]]", "[[Repos/cod-platform-web/CLAUDE]]", "[[Repos/cod-platform-ai/CLAUDE]]"]
---

# Increment 0 — Foundation

> All four repos exist, scaffolded with their stacks, and run together via `docker compose up`. Mock auth lets every API endpoint be called without real credentials. No business logic. The next coder can start Increment 1 the moment this is done.

## Goal

A new dev clones the four repos, runs `docker compose up` from `cod-platform-infra`, and within minutes sees:

- All containers healthy
- `GET http://localhost:3000/health` (API) → `{ status: "ok", service: "cod-platform-api", ... }`
- `GET http://localhost:3001` (Web) → page renders "OK"
- `GET http://localhost:8000/health` (AI) → `{ status: "ok", service: "cod-platform-ai", ... }`
- `GET http://localhost:3000/me` with header `x-mock-store-id: <seeded-store-uuid>` → returns the seeded owner context

That is the entirety of Increment 0. No business endpoints. No real auth. Just the rails.

## Decisions Locked

| Decision | Value |
|----------|-------|
| Package manager (TS repos) | npm |
| Node version | 22 LTS |
| NestJS scaffold | bare `nest new` |
| Next.js scaffold | App Router, TypeScript, Tailwind, ESLint, `src/` directory, no Turbopack |
| Python version | 3.12 |
| FastAPI tooling | uv |
| Postgres version | 16 |
| Redis version | 7 |
| Health check shape | `{ status, service, version, timestamp }` — same across all 3 services |
| Mock auth header | `x-mock-store-id` |
| Auth port shape | headers in, `AuthContext \| null` out |
| Dockerfile per service | from day one, multi-stage, pinned base image tags |

## Repos Affected

All four. Increment 0 is the only increment that touches every repo at once — afterwards each increment usually has one primary repo.

```
Repos/cod-platform-infra/   ← orchestration, env, compose
Repos/cod-platform-api/     ← NestJS + Prisma + AuthPort + mock seed
Repos/cod-platform-web/     ← Next.js placeholder page
Repos/cod-platform-ai/      ← FastAPI stub
```

---

## Scope per Repo

### `cod-platform-infra`

**Files created:**
```
cod-platform-infra/
├── README.md                ← purpose, how to run, environment variables overview
├── CLAUDE.md                ← already exists from vault sync
├── .env.example             ← exhaustive list, every var documented
├── docker-compose.yml       ← canonical stack
├── docker-compose.dev.yml   ← dev overrides (hot reload, mock auth on, source mounts)
├── docker-compose.test.yml  ← test database overrides (separate db name, reset on up)
├── .gitignore
└── docs/
    └── superpowers/
        ├── specs/
        │   └── 2026-04-26-inc0-foundation-design.md   ← this file
        └── plans/                                      ← writing-plans output later
```

**`docker-compose.yml` services:**

```yaml
version: "3.9"

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER}"]
      interval: 5s
      timeout: 3s
      retries: 10

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10

  api:
    build: ../cod-platform-api
    environment:
      DATABASE_URL: ${DATABASE_URL}
      REDIS_URL: ${REDIS_URL}
      AUTH_ADAPTER: ${AUTH_ADAPTER}
      MOCK_STORE_ID: ${MOCK_STORE_ID}
      MOCK_USER_ID: ${MOCK_USER_ID}
      NODE_ENV: ${NODE_ENV}
      PORT: 3000
    ports: ["3000:3000"]
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
      interval: 10s
      timeout: 3s
      retries: 5

  web:
    build: ../cod-platform-web
    environment:
      NEXT_PUBLIC_API_URL: http://localhost:3000
      NODE_ENV: ${NODE_ENV}
    ports: ["3001:3001"]
    depends_on:
      api:
        condition: service_healthy

  ai:
    build: ../cod-platform-ai
    environment:
      PORT: 8000
    ports: ["8000:8000"]
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8000/health"]
      interval: 10s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

**`docker-compose.dev.yml` overrides:**
- Each service gets a bind mount of its source directory for hot reload
- `AUTH_ADAPTER=mock` set for the API
- `NODE_ENV=development`
- `command:` overrides to use dev servers (`npm run start:dev` / `next dev` / `uv run uvicorn ... --reload`)

**`.env.example`:**

```env
# Postgres
POSTGRES_DB=cod_platform
POSTGRES_USER=cod_user
POSTGRES_PASSWORD=changeme

# Connection strings (used by services)
DATABASE_URL=postgresql://cod_user:changeme@postgres:5432/cod_platform
REDIS_URL=redis://redis:6379

# Auth
AUTH_ADAPTER=mock                                   # mock | jwt
MOCK_STORE_ID=00000000-0000-0000-0000-000000000001
MOCK_USER_ID=00000000-0000-0000-0000-000000000002

# Runtime
NODE_ENV=development
```

`.env.example` is the canonical list. Any new env var added by any service in any future increment MUST be added here in the same PR.

---

### `cod-platform-api`

**Scaffold command (one-time, from `cod-platform-api/`):**
```
npx -y @nestjs/cli@latest new . --package-manager npm --strict
```

**Initial directory layout (after scaffold + Increment 0 work):**
```
cod-platform-api/
├── README.md
├── CLAUDE.md
├── Dockerfile                      ← multi-stage
├── .dockerignore
├── package.json
├── package-lock.json
├── tsconfig.json
├── nest-cli.json
├── prisma/
│   └── schema.prisma               ← provider only; no models in Increment 0
├── src/
│   ├── main.ts                     ← bootstrap, port from env
│   ├── app.module.ts               ← imports HealthModule, AuthModule, PrismaModule
│   ├── prisma/
│   │   ├── prisma.module.ts
│   │   └── prisma.service.ts       ← Prisma client wrapper
│   ├── health/
│   │   ├── health.module.ts
│   │   └── health.controller.ts    ← GET /health
│   └── auth/
│       ├── auth.module.ts
│       ├── auth.types.ts           ← AuthContext, AuthInput, StoreMemberRole stub
│       ├── auth.port.ts            ← AuthPort interface
│       ├── auth.token.ts           ← AUTH_PORT injection token
│       ├── adapters/
│       │   ├── mock-auth.adapter.ts
│       │   └── jwt-auth.adapter.ts ← stub: throws "not implemented" — wired in Increment 8
│       ├── auth.guard.ts           ← global guard, calls AuthPort, attaches context to request
│       └── decorators/
│           └── current-context.decorator.ts   ← @CurrentContext() param decorator
├── test/
│   └── health.e2e-spec.ts
└── docs/
    └── superpowers/
        ├── specs/                  ← future increment specs (Increment 1+)
        └── plans/                  ← future increment plans
```

**`prisma/schema.prisma` for Increment 0:**

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

No models yet. Increment 2 adds the schema. Migration tooling and client generation are wired now so Increment 1 can run unit tests against generated types if needed.

**`src/auth/auth.port.ts`:**

```ts
import { AuthContext } from "./auth.types"

export interface AuthPort {
  resolveContext(headers: Record<string, string>): Promise<AuthContext | null>
}
```

**`src/auth/auth.types.ts`:**

```ts
export type StoreMemberRole = "OWNER" | "MANAGER" | "ASSISTANT" | "VIEWER"

export interface AuthContext {
  userId: string
  storeId: string
  role: StoreMemberRole
}
```

**`src/auth/auth.token.ts`:**

```ts
export const AUTH_PORT = Symbol("AUTH_PORT")
```

**`src/auth/adapters/mock-auth.adapter.ts`:**

```ts
import { Injectable } from "@nestjs/common"
import { AuthPort } from "../auth.port"
import { AuthContext } from "../auth.types"

@Injectable()
export class MockAuthAdapter implements AuthPort {
  async resolveContext(headers: Record<string, string>): Promise<AuthContext | null> {
    const storeId = headers["x-mock-store-id"]
    if (!storeId) return null
    return {
      userId: process.env.MOCK_USER_ID!,
      storeId,
      role: "OWNER",
    }
  }
}
```

The mock adapter does NOT validate that the store exists in the DB during Increment 0 (no schema yet). Validation is added in Increment 2 once the schema is live.

**`src/auth/adapters/jwt-auth.adapter.ts`:**

```ts
import { Injectable } from "@nestjs/common"
import { AuthPort } from "../auth.port"
import { AuthContext } from "../auth.types"

@Injectable()
export class JwtAuthAdapter implements AuthPort {
  async resolveContext(_headers: Record<string, string>): Promise<AuthContext | null> {
    throw new Error("JwtAuthAdapter not implemented — Increment 8")
  }
}
```

Stub only. Implemented in Increment 8.

**`src/auth/auth.module.ts`:**

```ts
import { Module } from "@nestjs/common"
import { AUTH_PORT } from "./auth.token"
import { MockAuthAdapter } from "./adapters/mock-auth.adapter"
import { JwtAuthAdapter } from "./adapters/jwt-auth.adapter"
import { AuthGuard } from "./auth.guard"
import { APP_GUARD } from "@nestjs/core"

const adapter = process.env.AUTH_ADAPTER === "jwt" ? JwtAuthAdapter : MockAuthAdapter

@Module({
  providers: [
    MockAuthAdapter,
    JwtAuthAdapter,
    { provide: AUTH_PORT, useExisting: adapter },
    { provide: APP_GUARD, useClass: AuthGuard },
  ],
  exports: [AUTH_PORT],
})
export class AuthModule {}
```

**`src/auth/auth.guard.ts`:**

Reads request headers, calls `AuthPort.resolveContext`, attaches `request.authContext` if non-null, throws 401 if null. Single global guard via `APP_GUARD`. The health endpoint opts out via a `@Public()` decorator (added in this increment for symmetry).

**`src/health/health.controller.ts`:**

```ts
import { Controller, Get } from "@nestjs/common"
import { Public } from "../auth/decorators/public.decorator"

@Controller("health")
export class HealthController {
  @Public()
  @Get()
  health() {
    return {
      status: "ok",
      service: "cod-platform-api",
      version: process.env.npm_package_version ?? "0.0.0",
      timestamp: new Date().toISOString(),
    }
  }
}
```

**Endpoints in Increment 0:**
- `GET /health` (public)
- `GET /me` (protected) — returns the resolved AuthContext for the request, used as the smoke test that the auth port wiring works

**`Dockerfile` (multi-stage):**

```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:22-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npx prisma generate
RUN npm run build

FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY --from=build /app/prisma ./prisma
COPY package.json ./
EXPOSE 3000
USER node
CMD ["node", "dist/main.js"]
```

---

### `cod-platform-web`

**Scaffold command:**
```
npx create-next-app@latest . --typescript --tailwind --eslint --app --src-dir --import-alias "@/*" --no-turbo
```

**Increment 0 deliverable:**
- `src/app/page.tsx` renders a centered "OK — cod-platform-web" with a reference to the API health
- `src/app/health/page.tsx` returns the same shape as the API (renders as JSON-looking text in the page for parity, even though it's not a real JSON endpoint)
- `next.config.js` minimal
- `Dockerfile` multi-stage, runs `next start` on port 3001

That's it for Increment 0. The dashboard pages come in Increment 9.

**`Dockerfile`:**

```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:22-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/.next ./.next
COPY --from=build /app/public ./public
COPY --from=deps /app/node_modules ./node_modules
COPY package.json ./
EXPOSE 3001
USER node
CMD ["npx", "next", "start", "-p", "3001"]
```

---

### `cod-platform-ai`

**Scaffold (manual — uv-based):**

```
cod-platform-ai/
├── README.md
├── CLAUDE.md
├── Dockerfile
├── .dockerignore
├── pyproject.toml          ← uv-managed, fastapi + uvicorn deps
├── uv.lock
├── src/
│   └── cod_platform_ai/
│       ├── __init__.py
│       ├── main.py         ← FastAPI app with /health
│       └── settings.py     ← pydantic-settings, reads env
└── docs/
    └── superpowers/
        ├── specs/
        └── plans/
```

**`src/cod_platform_ai/main.py`:**

```python
from datetime import datetime, timezone
from fastapi import FastAPI

app = FastAPI(title="cod-platform-ai")

@app.get("/health")
def health():
    return {
        "status": "ok",
        "service": "cod-platform-ai",
        "version": "0.0.0",
        "timestamp": datetime.now(timezone.utc).isoformat(),
    }
```

That is the entirety of `cod-platform-ai` for Phase 1. Real implementation begins in Phase 3.

**`Dockerfile`:**

```dockerfile
FROM python:3.12-slim AS base
WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1

FROM base AS deps
RUN pip install --no-cache-dir uv
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

FROM base AS runner
WORKDIR /app
COPY --from=deps /app/.venv /app/.venv
COPY src ./src
ENV PATH="/app/.venv/bin:$PATH"
EXPOSE 8000
CMD ["uvicorn", "cod_platform_ai.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## Mock Auth Smoke Flow

Once the stack is running, the smoke test for auth wiring:

```bash
# 1. Health (public endpoint, no auth needed)
curl http://localhost:3000/health
# → { "status": "ok", "service": "cod-platform-api", ... }

# 2. /me without header (mock adapter returns null → guard returns 401)
curl -i http://localhost:3000/me
# → HTTP 401

# 3. /me with mock header
curl -H "x-mock-store-id: 00000000-0000-0000-0000-000000000001" \
     http://localhost:3000/me
# → { "userId": "...", "storeId": "00000000-...", "role": "OWNER" }
```

The same flow with `AUTH_ADAPTER=jwt` (set in env) returns 500 on every endpoint because `JwtAuthAdapter.resolveContext` throws. That's intentional — proves the env switch actually changes behavior.

## Tests (Increment 0 minimum)

| Test | Repo | What it covers |
|------|------|----------------|
| `health.e2e-spec.ts` | api | `GET /health` returns 200 with the right shape |
| `me.e2e-spec.ts` | api | `GET /me` with mock header returns OWNER context; without returns 401 |
| `auth.guard.spec.ts` | api | Guard calls AuthPort, attaches context, allows `@Public()` |
| `mock-auth.adapter.spec.ts` | api | Returns OWNER context when header present, null otherwise |
| `compose smoke` | infra | Manual: `docker compose up`, hit all 3 health endpoints, assert 200 |

No unit tests in `cod-platform-web` or `cod-platform-ai` for Increment 0 — there's no logic to test yet.

## Done Criteria

All of these are necessary; none can be deferred:

- [ ] All four repos exist with `CLAUDE.md`, `README.md`, and proper `.gitignore`
- [ ] Each TypeScript repo has `package.json` with pinned Node 22 engines field
- [ ] `cod-platform-ai` has `pyproject.toml` pinned to Python 3.12
- [ ] Each service has a multi-stage `Dockerfile` and `.dockerignore`
- [ ] `cod-platform-infra/.env.example` lists every variable any service uses
- [ ] `docker-compose.yml` brings up postgres + redis + api + web + ai with healthchecks
- [ ] `docker-compose.dev.yml` provides hot reload for all three services
- [ ] `GET /health` returns 200 with `{ status, service, version, timestamp }` on all three services
- [ ] `cod-platform-api` Prisma is initialized; `npx prisma migrate status` runs without error against the dockerized Postgres
- [ ] `AuthPort`, `MockAuthAdapter`, and `JwtAuthAdapter` (stub) exist; `AuthGuard` is wired globally; `@Public()` works for `/health`
- [ ] `GET /me` with mock header returns owner context; without header returns 401
- [ ] All listed unit/e2e tests pass in CI-equivalent (locally via `npm test`)
- [ ] Manual compose smoke documented in each repo's README

## Out of Scope (explicit)

These are intentionally NOT in Increment 0. If they tempt you, document them and stop.

- Any business endpoint (orders, products, batches, etc.) — Increment 3+
- Prisma models — Increment 2
- RLS policies — Increment 2 (no models to apply them to yet)
- Real JWT — Increment 8
- StoreMember + permission matrix enforcement — Increment 8
- Production deployment — Increment 10
- Frontend dashboard pages beyond a placeholder — Increment 9
- AI service logic — Phase 3
- Logging beyond default NestJS logger — added when needed by a real feature
- CI/CD pipelines — added in Increment 10

## Open Questions (resolved before plan)

None. Every decision is locked above.

## Backlinks

- [[COD Platform/Roadmap/Phase 1 Design Spec]] — parent spec; Increment 0 is the first row in the increment map
- [[COD Platform/Architecture/Tech Stack]] — service architecture this scaffolds
- [[Repos/cod-platform-infra/CLAUDE]] — primary repo for this increment
- [[Repos/cod-platform-api/CLAUDE]] — secondary repo (most code in this increment)
- [[Repos/cod-platform-web/CLAUDE]] — secondary repo
- [[Repos/cod-platform-ai/CLAUDE]] — secondary repo (stub only)
- [[COD Platform/Rules/Development Rules]] — workflow pattern this follows
