# Increment 0 — Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** All four COD Platform repos exist as real git repos, scaffolded with their stacks, and run together via `docker compose up`. Mock auth lets every API endpoint be called without real credentials. No business logic — just the rails.

**Architecture:** Four separate repos (`cod-platform-infra`, `cod-platform-api`, `cod-platform-web`, `cod-platform-ai`). Infra orchestrates the others via docker-compose. The API uses a port/adapter pattern for auth so dev runs without real JWT — `MockAuthAdapter` reads a hardcoded store ID from a header. `JwtAuthAdapter` is a stub thrown into Increment 8.

**Tech Stack:** NestJS + Prisma (TS), Next.js App Router (TS, Tailwind), FastAPI + uv (Python 3.12), PostgreSQL 16, Redis 7, Docker Compose.

**Spec:** [`2026-04-26-inc0-foundation-design.md`](../specs/2026-04-26-inc0-foundation-design.md)

**Working directory:** the engineer runs commands from inside the `Repos/<repo-name>/` folder for each repo. All four folders already exist with `CLAUDE.md` and `docs/` from the brainstorming phase. The plan turns each into an actual git repo with code.

---

## Task 1: Initialize the four repos as git repos

**Files:**
- Modify (init): `Repos/cod-platform-infra/`, `Repos/cod-platform-api/`, `Repos/cod-platform-web/`, `Repos/cod-platform-ai/`

- [ ] **Step 1: Initialize git in cod-platform-infra**

```bash
cd Repos/cod-platform-infra
git init -b main
```

- [ ] **Step 2: Initialize git in cod-platform-api**

```bash
cd ../cod-platform-api
git init -b main
```

- [ ] **Step 3: Initialize git in cod-platform-web**

```bash
cd ../cod-platform-web
git init -b main
```

- [ ] **Step 4: Initialize git in cod-platform-ai**

```bash
cd ../cod-platform-ai
git init -b main
```

- [ ] **Step 5: Verify all four are git repos**

```bash
cd ..
for d in cod-platform-infra cod-platform-api cod-platform-web cod-platform-ai; do
  echo "=== $d ===" && git -C "$d" status --short
done
```

Expected: each prints "On branch main\nNo commits yet\nUntracked files: CLAUDE.md ...".

---

## Task 2: Add baseline gitignore + README to cod-platform-infra

**Files:**
- Create: `Repos/cod-platform-infra/.gitignore`
- Create: `Repos/cod-platform-infra/README.md`

- [ ] **Step 1: Create `.gitignore`**

Write `Repos/cod-platform-infra/.gitignore`:

```gitignore
# Env
.env
.env.local
.env.*.local

# OS
.DS_Store
Thumbs.db

# Editors
.vscode/
.idea/

# Logs
*.log
```

- [ ] **Step 2: Create `README.md`**

Write `Repos/cod-platform-infra/README.md`:

```markdown
# cod-platform-infra

Infrastructure and orchestration repo for the COD Platform.

## Quick start

1. Copy `.env.example` to `.env` and adjust if needed.
2. Make sure the sibling repos exist next to this one:
   - `../cod-platform-api`
   - `../cod-platform-web`
   - `../cod-platform-ai`
3. Run the stack:

       docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build

4. Smoke test:

       curl http://localhost:3000/health
       curl http://localhost:3001
       curl http://localhost:8000/health
       curl -H "x-mock-store-id: 00000000-0000-0000-0000-000000000001" http://localhost:3000/me

## Layout

- `docker-compose.yml`           Canonical stack
- `docker-compose.dev.yml`       Dev overrides (hot reload, mock auth)
- `docker-compose.test.yml`      Test database overrides
- `.env.example`                 Canonical env var list
- `docs/superpowers/specs/`      Increment specs
- `docs/superpowers/plans/`      Increment plans

See `CLAUDE.md` for the session protocol every agent and coder must follow.
```

- [ ] **Step 3: Commit**

```bash
cd Repos/cod-platform-infra
git add .gitignore README.md CLAUDE.md docs/
git commit -m "chore: initial infra repo with gitignore and readme"
```

---

## Task 3: Create `.env.example` in cod-platform-infra

**Files:**
- Create: `Repos/cod-platform-infra/.env.example`

- [ ] **Step 1: Create `.env.example`**

Write `Repos/cod-platform-infra/.env.example`:

```env
# Postgres
POSTGRES_DB=cod_platform
POSTGRES_USER=cod_user
POSTGRES_PASSWORD=changeme

# Connection strings (used by services inside the docker network)
DATABASE_URL=postgresql://cod_user:changeme@postgres:5432/cod_platform
REDIS_URL=redis://redis:6379

# Auth
AUTH_ADAPTER=mock
MOCK_STORE_ID=00000000-0000-0000-0000-000000000001
MOCK_USER_ID=00000000-0000-0000-0000-000000000002

# Runtime
NODE_ENV=development
```

- [ ] **Step 2: Commit**

```bash
git add .env.example
git commit -m "chore: add canonical env example"
```

---

## Task 4: Scaffold cod-platform-api with `nest new`

**Files:**
- Create: `Repos/cod-platform-api/package.json`, `tsconfig.json`, `nest-cli.json`, `src/`, etc. (via NestJS CLI)

- [ ] **Step 1: Run `nest new`**

```bash
cd ../cod-platform-api
npx -y @nestjs/cli@latest new . --package-manager npm --strict --skip-git
```

When prompted to overwrite the existing `CLAUDE.md` or `docs/` folder, choose to keep the existing files.

Expected: NestJS scaffold completes; `package.json`, `src/main.ts`, `src/app.module.ts`, `src/app.controller.ts`, `src/app.service.ts` exist.

- [ ] **Step 2: Pin Node version in `package.json`**

Open `package.json` and add an `engines` field:

```json
{
  "engines": {
    "node": ">=22.0.0 <23"
  }
}
```

- [ ] **Step 3: Remove the boilerplate `app.controller.ts` and `app.service.ts`**

```bash
rm src/app.controller.ts src/app.controller.spec.ts src/app.service.ts
```

- [ ] **Step 4: Update `src/app.module.ts` to remove references to deleted files**

Replace `src/app.module.ts` with:

```ts
import { Module } from "@nestjs/common"

@Module({
  imports: [],
  controllers: [],
  providers: [],
})
export class AppModule {}
```

- [ ] **Step 5: Update `src/main.ts` to read PORT from env**

Replace `src/main.ts` with:

```ts
import { NestFactory } from "@nestjs/core"
import { AppModule } from "./app.module"

async function bootstrap() {
  const app = await NestFactory.create(AppModule)
  const port = Number(process.env.PORT ?? 3000)
  await app.listen(port)
}
bootstrap()
```

- [ ] **Step 6: Verify it builds**

```bash
npm run build
```

Expected: build succeeds with no errors. `dist/main.js` exists.

- [ ] **Step 7: Commit**

```bash
git add .
git commit -m "feat: scaffold nestjs api with bare modules"
```

---

## Task 5: Add Prisma to cod-platform-api

**Files:**
- Create: `Repos/cod-platform-api/prisma/schema.prisma`
- Create: `Repos/cod-platform-api/src/prisma/prisma.module.ts`
- Create: `Repos/cod-platform-api/src/prisma/prisma.service.ts`
- Modify: `Repos/cod-platform-api/src/app.module.ts`

- [ ] **Step 1: Install Prisma**

```bash
npm install --save-dev prisma
npm install @prisma/client
```

- [ ] **Step 2: Initialize Prisma**

```bash
npx prisma init --datasource-provider postgresql
```

This creates `prisma/schema.prisma` and a default `.env` (we'll delete it — env comes from compose).

- [ ] **Step 3: Replace `prisma/schema.prisma`**

Replace contents with:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

- [ ] **Step 4: Delete the auto-created `.env` (env comes from compose)**

```bash
rm -f .env
```

- [ ] **Step 5: Add `.env` and `node_modules` to `.gitignore` (create if absent)**

Write `Repos/cod-platform-api/.gitignore`:

```gitignore
# Node
node_modules/
dist/
*.log

# Env
.env
.env.local
.env.*.local

# OS
.DS_Store
Thumbs.db

# Editors
.vscode/
.idea/

# Prisma
prisma/migrations/dev.db*
```

- [ ] **Step 6: Create `src/prisma/prisma.service.ts`**

```ts
import { Injectable, OnModuleInit, OnModuleDestroy } from "@nestjs/common"
import { PrismaClient } from "@prisma/client"

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit() {
    await this.$connect()
  }

  async onModuleDestroy() {
    await this.$disconnect()
  }
}
```

- [ ] **Step 7: Create `src/prisma/prisma.module.ts`**

```ts
import { Global, Module } from "@nestjs/common"
import { PrismaService } from "./prisma.service"

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

- [ ] **Step 8: Wire PrismaModule into `app.module.ts`**

Replace `src/app.module.ts`:

```ts
import { Module } from "@nestjs/common"
import { PrismaModule } from "./prisma/prisma.module"

@Module({
  imports: [PrismaModule],
  controllers: [],
  providers: [],
})
export class AppModule {}
```

- [ ] **Step 9: Generate Prisma client**

```bash
npx prisma generate
```

Expected: `Generated Prisma Client` message; no errors.

- [ ] **Step 10: Verify build still succeeds**

```bash
npm run build
```

Expected: build succeeds.

- [ ] **Step 11: Commit**

```bash
git add .gitignore prisma/ src/prisma/ src/app.module.ts package.json package-lock.json
git commit -m "feat: add prisma client with global module"
```

---

## Task 6: Create the AuthPort interface + types

**Files:**
- Create: `Repos/cod-platform-api/src/auth/auth.types.ts`
- Create: `Repos/cod-platform-api/src/auth/auth.port.ts`
- Create: `Repos/cod-platform-api/src/auth/auth.token.ts`

- [ ] **Step 1: Create `src/auth/auth.types.ts`**

```ts
export type StoreMemberRole = "OWNER" | "MANAGER" | "ASSISTANT" | "VIEWER"

export interface AuthContext {
  userId: string
  storeId: string
  role: StoreMemberRole
}
```

- [ ] **Step 2: Create `src/auth/auth.port.ts`**

```ts
import { AuthContext } from "./auth.types"

export interface AuthPort {
  resolveContext(headers: Record<string, string>): Promise<AuthContext | null>
}
```

- [ ] **Step 3: Create `src/auth/auth.token.ts`**

```ts
export const AUTH_PORT = Symbol("AUTH_PORT")
```

- [ ] **Step 4: Verify build**

```bash
npm run build
```

Expected: build succeeds.

- [ ] **Step 5: Commit**

```bash
git add src/auth/
git commit -m "feat: define AuthPort interface and types"
```

---

## Task 7: Implement MockAuthAdapter (TDD)

**Files:**
- Create: `Repos/cod-platform-api/src/auth/adapters/mock-auth.adapter.spec.ts`
- Create: `Repos/cod-platform-api/src/auth/adapters/mock-auth.adapter.ts`

- [ ] **Step 1: Write the failing test**

Create `src/auth/adapters/mock-auth.adapter.spec.ts`:

```ts
import { MockAuthAdapter } from "./mock-auth.adapter"

describe("MockAuthAdapter", () => {
  let originalEnv: NodeJS.ProcessEnv

  beforeEach(() => {
    originalEnv = { ...process.env }
    process.env.MOCK_USER_ID = "00000000-0000-0000-0000-000000000002"
  })

  afterEach(() => {
    process.env = originalEnv
  })

  it("returns OWNER context when x-mock-store-id header is present", async () => {
    const adapter = new MockAuthAdapter()
    const ctx = await adapter.resolveContext({
      "x-mock-store-id": "00000000-0000-0000-0000-000000000001",
    })

    expect(ctx).toEqual({
      userId: "00000000-0000-0000-0000-000000000002",
      storeId: "00000000-0000-0000-0000-000000000001",
      role: "OWNER",
    })
  })

  it("returns null when header is absent", async () => {
    const adapter = new MockAuthAdapter()
    const ctx = await adapter.resolveContext({})
    expect(ctx).toBeNull()
  })
})
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npm test -- mock-auth.adapter
```

Expected: FAIL with "Cannot find module './mock-auth.adapter'".

- [ ] **Step 3: Implement `src/auth/adapters/mock-auth.adapter.ts`**

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

- [ ] **Step 4: Run the test to verify it passes**

```bash
npm test -- mock-auth.adapter
```

Expected: 2 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add src/auth/adapters/mock-auth.adapter.ts src/auth/adapters/mock-auth.adapter.spec.ts
git commit -m "feat: implement MockAuthAdapter with tests"
```

---

## Task 8: Implement JwtAuthAdapter stub

**Files:**
- Create: `Repos/cod-platform-api/src/auth/adapters/jwt-auth.adapter.ts`

- [ ] **Step 1: Create the stub**

Write `src/auth/adapters/jwt-auth.adapter.ts`:

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

- [ ] **Step 2: Verify build**

```bash
npm run build
```

Expected: build succeeds.

- [ ] **Step 3: Commit**

```bash
git add src/auth/adapters/jwt-auth.adapter.ts
git commit -m "feat: stub JwtAuthAdapter (implementation in increment 8)"
```

---

## Task 9: Add `@Public()` decorator + AuthGuard (TDD)

**Files:**
- Create: `Repos/cod-platform-api/src/auth/decorators/public.decorator.ts`
- Create: `Repos/cod-platform-api/src/auth/decorators/current-context.decorator.ts`
- Create: `Repos/cod-platform-api/src/auth/auth.guard.ts`
- Create: `Repos/cod-platform-api/src/auth/auth.guard.spec.ts`

- [ ] **Step 1: Create `src/auth/decorators/public.decorator.ts`**

```ts
import { SetMetadata } from "@nestjs/common"

export const IS_PUBLIC_KEY = "isPublic"
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true)
```

- [ ] **Step 2: Create `src/auth/decorators/current-context.decorator.ts`**

```ts
import { createParamDecorator, ExecutionContext } from "@nestjs/common"
import { AuthContext } from "../auth.types"

export const CurrentContext = createParamDecorator(
  (_data: unknown, ctx: ExecutionContext): AuthContext => {
    const req = ctx.switchToHttp().getRequest()
    return req.authContext
  },
)
```

- [ ] **Step 3: Write the failing guard test**

Create `src/auth/auth.guard.spec.ts`:

```ts
import { ExecutionContext, UnauthorizedException } from "@nestjs/common"
import { Reflector } from "@nestjs/core"
import { AuthGuard } from "./auth.guard"
import { AuthPort } from "./auth.port"
import { AuthContext } from "./auth.types"
import { IS_PUBLIC_KEY } from "./decorators/public.decorator"

const mockExecutionContext = (headers: Record<string, string>): ExecutionContext => {
  const req: any = { headers, authContext: undefined }
  return {
    switchToHttp: () => ({ getRequest: () => req }),
    getHandler: () => ({}) as any,
    getClass: () => ({}) as any,
  } as ExecutionContext
}

describe("AuthGuard", () => {
  it("allows public routes without context", async () => {
    const reflector = { getAllAndOverride: jest.fn().mockReturnValue(true) } as unknown as Reflector
    const port: AuthPort = { resolveContext: jest.fn() }
    const guard = new AuthGuard(reflector, port)

    const ctx = mockExecutionContext({})
    await expect(guard.canActivate(ctx)).resolves.toBe(true)
    expect(port.resolveContext).not.toHaveBeenCalled()
  })

  it("attaches context and allows when adapter returns context", async () => {
    const reflector = { getAllAndOverride: jest.fn().mockReturnValue(false) } as unknown as Reflector
    const ctxData: AuthContext = { userId: "u", storeId: "s", role: "OWNER" }
    const port: AuthPort = { resolveContext: jest.fn().mockResolvedValue(ctxData) }
    const guard = new AuthGuard(reflector, port)

    const execCtx = mockExecutionContext({ "x-mock-store-id": "s" })
    await expect(guard.canActivate(execCtx)).resolves.toBe(true)
    expect(execCtx.switchToHttp().getRequest().authContext).toEqual(ctxData)
  })

  it("throws Unauthorized when adapter returns null on protected route", async () => {
    const reflector = { getAllAndOverride: jest.fn().mockReturnValue(false) } as unknown as Reflector
    const port: AuthPort = { resolveContext: jest.fn().mockResolvedValue(null) }
    const guard = new AuthGuard(reflector, port)

    const ctx = mockExecutionContext({})
    await expect(guard.canActivate(ctx)).rejects.toThrow(UnauthorizedException)
  })
})
```

- [ ] **Step 4: Run the test to verify it fails**

```bash
npm test -- auth.guard.spec
```

Expected: FAIL with "Cannot find module './auth.guard'".

- [ ] **Step 5: Implement `src/auth/auth.guard.ts`**

```ts
import {
  CanActivate,
  ExecutionContext,
  Inject,
  Injectable,
  UnauthorizedException,
} from "@nestjs/common"
import { Reflector } from "@nestjs/core"
import { AuthPort } from "./auth.port"
import { AUTH_PORT } from "./auth.token"
import { IS_PUBLIC_KEY } from "./decorators/public.decorator"

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(
    private readonly reflector: Reflector,
    @Inject(AUTH_PORT) private readonly authPort: AuthPort,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ])
    if (isPublic) return true

    const req = context.switchToHttp().getRequest()
    const authContext = await this.authPort.resolveContext(req.headers ?? {})
    if (!authContext) throw new UnauthorizedException()

    req.authContext = authContext
    return true
  }
}
```

- [ ] **Step 6: Run the test to verify it passes**

```bash
npm test -- auth.guard.spec
```

Expected: 3 tests PASS.

- [ ] **Step 7: Commit**

```bash
git add src/auth/
git commit -m "feat: add @Public decorator, @CurrentContext, and AuthGuard"
```

---

## Task 10: Wire AuthModule with adapter selection

**Files:**
- Create: `Repos/cod-platform-api/src/auth/auth.module.ts`
- Modify: `Repos/cod-platform-api/src/app.module.ts`

- [ ] **Step 1: Create `src/auth/auth.module.ts`**

```ts
import { Module } from "@nestjs/common"
import { APP_GUARD } from "@nestjs/core"
import { AUTH_PORT } from "./auth.token"
import { MockAuthAdapter } from "./adapters/mock-auth.adapter"
import { JwtAuthAdapter } from "./adapters/jwt-auth.adapter"
import { AuthGuard } from "./auth.guard"

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

- [ ] **Step 2: Wire AuthModule into `app.module.ts`**

Replace `src/app.module.ts`:

```ts
import { Module } from "@nestjs/common"
import { PrismaModule } from "./prisma/prisma.module"
import { AuthModule } from "./auth/auth.module"

@Module({
  imports: [PrismaModule, AuthModule],
  controllers: [],
  providers: [],
})
export class AppModule {}
```

- [ ] **Step 3: Verify build**

```bash
npm run build
```

Expected: build succeeds.

- [ ] **Step 4: Commit**

```bash
git add src/auth/auth.module.ts src/app.module.ts
git commit -m "feat: wire AuthModule with mock/jwt adapter selection via env"
```

---

## Task 11: Add HealthModule + GET /health

**Files:**
- Create: `Repos/cod-platform-api/src/health/health.module.ts`
- Create: `Repos/cod-platform-api/src/health/health.controller.ts`
- Create: `Repos/cod-platform-api/test/health.e2e-spec.ts`
- Modify: `Repos/cod-platform-api/src/app.module.ts`

- [ ] **Step 1: Create `src/health/health.controller.ts`**

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

- [ ] **Step 2: Create `src/health/health.module.ts`**

```ts
import { Module } from "@nestjs/common"
import { HealthController } from "./health.controller"

@Module({
  controllers: [HealthController],
})
export class HealthModule {}
```

- [ ] **Step 3: Wire HealthModule into `app.module.ts`**

Replace `src/app.module.ts`:

```ts
import { Module } from "@nestjs/common"
import { PrismaModule } from "./prisma/prisma.module"
import { AuthModule } from "./auth/auth.module"
import { HealthModule } from "./health/health.module"

@Module({
  imports: [PrismaModule, AuthModule, HealthModule],
  controllers: [],
  providers: [],
})
export class AppModule {}
```

- [ ] **Step 4: Write e2e test `test/health.e2e-spec.ts`**

```ts
import { Test, TestingModule } from "@nestjs/testing"
import { INestApplication } from "@nestjs/common"
import * as request from "supertest"
import { AppModule } from "../src/app.module"

describe("GET /health (e2e)", () => {
  let app: INestApplication

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile()

    app = moduleFixture.createNestApplication()
    await app.init()
  })

  afterAll(async () => {
    await app.close()
  })

  it("returns 200 with the right shape", async () => {
    const res = await request(app.getHttpServer()).get("/health")
    expect(res.status).toBe(200)
    expect(res.body).toMatchObject({
      status: "ok",
      service: "cod-platform-api",
    })
    expect(res.body.version).toBeDefined()
    expect(res.body.timestamp).toBeDefined()
  })
})
```

- [ ] **Step 5: Run the e2e test**

```bash
npm run test:e2e -- health
```

Expected: 1 test PASS.

- [ ] **Step 6: Commit**

```bash
git add src/health/ src/app.module.ts test/health.e2e-spec.ts
git commit -m "feat: add /health endpoint marked @Public with e2e test"
```

---

## Task 12: Add MeModule + GET /me (auth smoke endpoint)

**Files:**
- Create: `Repos/cod-platform-api/src/me/me.module.ts`
- Create: `Repos/cod-platform-api/src/me/me.controller.ts`
- Create: `Repos/cod-platform-api/test/me.e2e-spec.ts`
- Modify: `Repos/cod-platform-api/src/app.module.ts`

- [ ] **Step 1: Create `src/me/me.controller.ts`**

```ts
import { Controller, Get } from "@nestjs/common"
import { CurrentContext } from "../auth/decorators/current-context.decorator"
import { AuthContext } from "../auth/auth.types"

@Controller("me")
export class MeController {
  @Get()
  me(@CurrentContext() ctx: AuthContext) {
    return ctx
  }
}
```

- [ ] **Step 2: Create `src/me/me.module.ts`**

```ts
import { Module } from "@nestjs/common"
import { MeController } from "./me.controller"

@Module({
  controllers: [MeController],
})
export class MeModule {}
```

- [ ] **Step 3: Wire MeModule into `app.module.ts`**

Replace `src/app.module.ts`:

```ts
import { Module } from "@nestjs/common"
import { PrismaModule } from "./prisma/prisma.module"
import { AuthModule } from "./auth/auth.module"
import { HealthModule } from "./health/health.module"
import { MeModule } from "./me/me.module"

@Module({
  imports: [PrismaModule, AuthModule, HealthModule, MeModule],
  controllers: [],
  providers: [],
})
export class AppModule {}
```

- [ ] **Step 4: Write e2e test `test/me.e2e-spec.ts`**

```ts
import { Test, TestingModule } from "@nestjs/testing"
import { INestApplication } from "@nestjs/common"
import * as request from "supertest"
import { AppModule } from "../src/app.module"

describe("GET /me (e2e)", () => {
  let app: INestApplication

  beforeAll(async () => {
    process.env.AUTH_ADAPTER = "mock"
    process.env.MOCK_USER_ID = "00000000-0000-0000-0000-000000000002"

    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile()

    app = moduleFixture.createNestApplication()
    await app.init()
  })

  afterAll(async () => {
    await app.close()
  })

  it("returns OWNER context when x-mock-store-id header is set", async () => {
    const res = await request(app.getHttpServer())
      .get("/me")
      .set("x-mock-store-id", "00000000-0000-0000-0000-000000000001")

    expect(res.status).toBe(200)
    expect(res.body).toEqual({
      userId: "00000000-0000-0000-0000-000000000002",
      storeId: "00000000-0000-0000-0000-000000000001",
      role: "OWNER",
    })
  })

  it("returns 401 when header is absent", async () => {
    const res = await request(app.getHttpServer()).get("/me")
    expect(res.status).toBe(401)
  })
})
```

- [ ] **Step 5: Run the e2e tests**

```bash
npm run test:e2e -- me
```

Expected: 2 tests PASS.

- [ ] **Step 6: Run all tests to confirm nothing broke**

```bash
npm test && npm run test:e2e
```

Expected: all tests PASS.

- [ ] **Step 7: Commit**

```bash
git add src/me/ src/app.module.ts test/me.e2e-spec.ts
git commit -m "feat: add /me endpoint as auth smoke; e2e covers 200 and 401"
```

---

## Task 13: Add multi-stage Dockerfile to cod-platform-api

**Files:**
- Create: `Repos/cod-platform-api/Dockerfile`
- Create: `Repos/cod-platform-api/.dockerignore`

- [ ] **Step 1: Create `.dockerignore`**

```dockerignore
node_modules
dist
.env
.env.*
.git
.gitignore
*.log
test
**/*.spec.ts
**/*.e2e-spec.ts
docs
```

- [ ] **Step 2: Create `Dockerfile`**

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
RUN apk add --no-cache wget
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY --from=build /app/prisma ./prisma
COPY package.json ./
EXPOSE 3000
USER node
CMD ["node", "dist/main.js"]
```

- [ ] **Step 3: Build the image to verify Dockerfile works**

```bash
docker build -t cod-platform-api:dev .
```

Expected: image builds successfully through all three stages.

- [ ] **Step 4: Commit**

```bash
git add Dockerfile .dockerignore
git commit -m "chore: multi-stage dockerfile for api"
```

---

## Task 14: Scaffold cod-platform-web with create-next-app

**Files:**
- Many (created by create-next-app)

- [ ] **Step 1: Run create-next-app**

```bash
cd ../cod-platform-web
npx create-next-app@latest . --typescript --tailwind --eslint --app --src-dir --import-alias "@/*" --no-turbo --use-npm
```

When prompted to overwrite `CLAUDE.md` or `docs/`, choose to keep existing files.

Expected: Next.js scaffold completes.

- [ ] **Step 2: Pin Node version in `package.json`**

Add to `package.json`:

```json
"engines": {
  "node": ">=22.0.0 <23"
}
```

- [ ] **Step 3: Replace `src/app/page.tsx` with placeholder**

```tsx
export default function Home() {
  return (
    <main className="flex min-h-screen items-center justify-center bg-slate-50">
      <div className="rounded-lg border border-slate-200 bg-white p-8 shadow-sm">
        <h1 className="text-2xl font-semibold">OK — cod-platform-web</h1>
        <p className="mt-2 text-sm text-slate-600">
          Phase 1 placeholder. Real dashboard arrives in Increment 9.
        </p>
      </div>
    </main>
  )
}
```

- [ ] **Step 4: Verify dev server starts**

```bash
npm run dev
```

Expected: server starts on port 3000 (default). Hit `http://localhost:3000` in a browser to confirm the placeholder renders. Stop with Ctrl+C.

- [ ] **Step 5: Verify build**

```bash
npm run build
```

Expected: build succeeds.

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "feat: scaffold next.js web with placeholder home page"
```

---

## Task 15: Add multi-stage Dockerfile to cod-platform-web

**Files:**
- Create: `Repos/cod-platform-web/Dockerfile`
- Create: `Repos/cod-platform-web/.dockerignore`

- [ ] **Step 1: Create `.dockerignore`**

```dockerignore
node_modules
.next
.env
.env.*
.git
.gitignore
*.log
docs
```

- [ ] **Step 2: Create `Dockerfile`**

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

- [ ] **Step 3: Build the image to verify**

```bash
docker build -t cod-platform-web:dev .
```

Expected: image builds successfully.

- [ ] **Step 4: Commit**

```bash
git add Dockerfile .dockerignore
git commit -m "chore: multi-stage dockerfile for web"
```

---

## Task 16: Scaffold cod-platform-ai with uv

**Files:**
- Create: `Repos/cod-platform-ai/pyproject.toml`
- Create: `Repos/cod-platform-ai/src/cod_platform_ai/__init__.py`
- Create: `Repos/cod-platform-ai/src/cod_platform_ai/main.py`
- Create: `Repos/cod-platform-ai/.gitignore`
- Create: `Repos/cod-platform-ai/README.md`

- [ ] **Step 1: Move into the repo and confirm uv is installed**

```bash
cd ../cod-platform-ai
uv --version
```

If uv is missing: `curl -LsSf https://astral.sh/uv/install.sh | sh` (or follow https://docs.astral.sh/uv/getting-started/installation/).

- [ ] **Step 2: Initialize the project with uv**

```bash
uv init --name cod-platform-ai --package --python 3.12
```

This creates `pyproject.toml` and a `src/cod_platform_ai/` package layout.

- [ ] **Step 3: Add FastAPI and uvicorn dependencies**

```bash
uv add fastapi uvicorn[standard] pydantic-settings
```

- [ ] **Step 4: Replace `src/cod_platform_ai/main.py`**

```python
from datetime import datetime, timezone
from fastapi import FastAPI

app = FastAPI(title="cod-platform-ai")


@app.get("/health")
def health() -> dict[str, str]:
    return {
        "status": "ok",
        "service": "cod-platform-ai",
        "version": "0.0.0",
        "timestamp": datetime.now(timezone.utc).isoformat(),
    }
```

- [ ] **Step 5: Create `.gitignore`**

```gitignore
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
build/
dist/
*.egg-info/
.venv/
venv/
env/

# uv
.uv-cache/

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db

# Env
.env
.env.local
```

- [ ] **Step 6: Create `README.md`**

```markdown
# cod-platform-ai

FastAPI service stub. Real implementation begins in Phase 3.

## Run locally

    uv sync
    uv run uvicorn cod_platform_ai.main:app --reload --port 8000

## Health check

    curl http://localhost:8000/health
```

- [ ] **Step 7: Verify it starts**

```bash
uv run uvicorn cod_platform_ai.main:app --port 8000 &
sleep 2
curl -sS http://localhost:8000/health
kill %1 2>/dev/null || true
```

Expected: JSON response with `"status": "ok"` and `"service": "cod-platform-ai"`.

- [ ] **Step 8: Commit**

```bash
git add .
git commit -m "feat: scaffold fastapi ai service stub with /health"
```

---

## Task 17: Add multi-stage Dockerfile to cod-platform-ai

**Files:**
- Create: `Repos/cod-platform-ai/Dockerfile`
- Create: `Repos/cod-platform-ai/.dockerignore`

- [ ] **Step 1: Create `.dockerignore`**

```dockerignore
.venv
__pycache__
.uv-cache
.env
.env.*
.git
.gitignore
*.log
docs
```

- [ ] **Step 2: Create `Dockerfile`**

```dockerfile
FROM python:3.12-slim AS base
WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    UV_COMPILE_BYTECODE=1

FROM base AS deps
RUN pip install --no-cache-dir uv
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

FROM base AS runner
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends wget && rm -rf /var/lib/apt/lists/*
COPY --from=deps /app/.venv /app/.venv
COPY src ./src
ENV PATH="/app/.venv/bin:$PATH"
EXPOSE 8000
CMD ["uvicorn", "cod_platform_ai.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

- [ ] **Step 3: Build the image**

```bash
docker build -t cod-platform-ai:dev .
```

Expected: image builds successfully.

- [ ] **Step 4: Commit**

```bash
git add Dockerfile .dockerignore
git commit -m "chore: multi-stage dockerfile for ai service"
```

---

## Task 18: Create `docker-compose.yml` in infra

**Files:**
- Create: `Repos/cod-platform-infra/docker-compose.yml`

- [ ] **Step 1: Create `docker-compose.yml`**

```yaml
version: "3.9"

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER}"]
      interval: 5s
      timeout: 3s
      retries: 10

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10

  api:
    build:
      context: ../cod-platform-api
    environment:
      DATABASE_URL: ${DATABASE_URL}
      REDIS_URL: ${REDIS_URL}
      AUTH_ADAPTER: ${AUTH_ADAPTER}
      MOCK_STORE_ID: ${MOCK_STORE_ID}
      MOCK_USER_ID: ${MOCK_USER_ID}
      NODE_ENV: ${NODE_ENV}
      PORT: 3000
    ports:
      - "3000:3000"
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
    build:
      context: ../cod-platform-web
    environment:
      NEXT_PUBLIC_API_URL: http://localhost:3000
      NODE_ENV: ${NODE_ENV}
    ports:
      - "3001:3001"
    depends_on:
      api:
        condition: service_healthy

  ai:
    build:
      context: ../cod-platform-ai
    environment:
      PORT: 8000
    ports:
      - "8000:8000"
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8000/health"]
      interval: 10s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

- [ ] **Step 2: Verify config syntax**

```bash
cd ../cod-platform-infra
cp .env.example .env
docker compose config > /dev/null
```

Expected: no errors. `docker compose config` validates and prints the resolved config.

- [ ] **Step 3: Commit**

```bash
git add docker-compose.yml
git commit -m "feat: canonical docker-compose with postgres redis api web ai"
```

---

## Task 19: Create `docker-compose.dev.yml` overrides

**Files:**
- Create: `Repos/cod-platform-infra/docker-compose.dev.yml`

- [ ] **Step 1: Create `docker-compose.dev.yml`**

```yaml
version: "3.9"

services:
  api:
    command: ["npm", "run", "start:dev"]
    volumes:
      - ../cod-platform-api/src:/app/src
      - ../cod-platform-api/prisma:/app/prisma
    environment:
      AUTH_ADAPTER: mock
      NODE_ENV: development

  web:
    command: ["npx", "next", "dev", "-p", "3001"]
    volumes:
      - ../cod-platform-web/src:/app/src
      - ../cod-platform-web/public:/app/public
    environment:
      NODE_ENV: development

  ai:
    command: ["uvicorn", "cod_platform_ai.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
    volumes:
      - ../cod-platform-ai/src:/app/src
```

- [ ] **Step 2: Validate combined config**

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml config > /dev/null
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add docker-compose.dev.yml
git commit -m "feat: dev compose overrides with hot reload and mock auth"
```

---

## Task 20: Full stack smoke test

**Files:** none (manual smoke)

- [ ] **Step 1: Bring up the full stack**

```bash
cd Repos/cod-platform-infra
cp .env.example .env  # if not already
docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build -d
```

Expected: all five containers start. `docker compose ps` shows `postgres`, `redis`, `api`, `web`, `ai` as running and healthy (postgres + redis + api + ai have healthchecks).

- [ ] **Step 2: Wait briefly, then smoke test API health**

```bash
sleep 10
curl -fsS http://localhost:3000/health | tee /dev/stderr | grep -q '"status":"ok"'
```

Expected: JSON response with `"status":"ok"` and the grep exits 0.

- [ ] **Step 3: Smoke test AI health**

```bash
curl -fsS http://localhost:8000/health | tee /dev/stderr | grep -q '"status":"ok"'
```

Expected: same.

- [ ] **Step 4: Smoke test web placeholder**

```bash
curl -fsS http://localhost:3001 | grep -q "OK — cod-platform-web"
```

Expected: HTML containing the placeholder text.

- [ ] **Step 5: Smoke test mock auth — without header (expect 401)**

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3000/me
```

Expected: `401`.

- [ ] **Step 6: Smoke test mock auth — with header (expect 200 + OWNER context)**

```bash
curl -fsS -H "x-mock-store-id: 00000000-0000-0000-0000-000000000001" \
  http://localhost:3000/me
```

Expected: JSON `{ "userId": "...", "storeId": "00000000-0000-0000-0000-000000000001", "role": "OWNER" }`.

- [ ] **Step 7: Tear down**

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml down -v
```

- [ ] **Step 8: Document the smoke result in infra README**

Append to `Repos/cod-platform-infra/README.md`:

```markdown
## Smoke check

Last verified: 2026-04-26 — Increment 0 done.

    docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build
    curl http://localhost:3000/health
    curl http://localhost:3001
    curl http://localhost:8000/health
    curl -H "x-mock-store-id: 00000000-0000-0000-0000-000000000001" http://localhost:3000/me
```

- [ ] **Step 9: Commit infra README update**

```bash
cd Repos/cod-platform-infra
git add README.md
git commit -m "docs: record increment 0 smoke result"
```

---

## Task 21: Mark Increment 0 done in the plan

**Files:**
- Modify: this plan file (`Repos/cod-platform-infra/docs/superpowers/plans/2026-04-26-inc0-foundation-plan.md`)

- [ ] **Step 1: Replace this final task block with a "DONE" marker by editing the bottom of the file**

Append to the bottom of this plan:

```markdown
---

## ✅ Increment 0 Complete

All tasks 1–20 done and verified by smoke test on 2026-04-26.
Next: write the Increment 1 spec (Financial Engine Core) in `Repos/cod-platform-api/docs/superpowers/specs/`.
```

- [ ] **Step 2: Commit**

```bash
git add docs/superpowers/plans/2026-04-26-inc0-foundation-plan.md
git commit -m "docs: mark increment 0 complete"
```

---

## Spec Coverage Verification

| Spec section | Covered by task(s) |
|--------------|---------------------|
| Decisions Locked table | Tasks 4, 5, 14, 16 (scaffold flavors) + Tasks 13, 15, 17 (Dockerfiles) |
| `cod-platform-infra` files (.env.example, compose) | Tasks 2, 3, 18, 19 |
| `cod-platform-api` Prisma init | Task 5 |
| `cod-platform-api` AuthPort + types | Task 6 |
| MockAuthAdapter | Task 7 |
| JwtAuthAdapter stub | Task 8 |
| AuthGuard + @Public + @CurrentContext | Task 9 |
| AuthModule with adapter selection | Task 10 |
| HealthController | Task 11 |
| /me endpoint | Task 12 |
| API Dockerfile | Task 13 |
| `cod-platform-web` scaffold + placeholder | Task 14 |
| Web Dockerfile | Task 15 |
| `cod-platform-ai` scaffold + /health | Task 16 |
| AI Dockerfile | Task 17 |
| docker-compose.yml | Task 18 |
| docker-compose.dev.yml | Task 19 |
| Mock auth smoke flow (3 curls) | Task 20 |
| Done criteria checklist | Tasks 4 (Node engines), 11/12 (e2e tests), 20 (smoke) |
| Test list (4 e2e + 1 manual) | Tasks 7, 9, 11, 12, 20 |

Every spec requirement has a task. No placeholders. Type names (`AuthPort`, `AuthContext`, `MockAuthAdapter`, `JwtAuthAdapter`, `AuthGuard`, `IS_PUBLIC_KEY`, `AUTH_PORT`, `Public`, `CurrentContext`) are consistent across tasks.
