# cod-platform-infra

Infrastructure and orchestration repo — the glue that runs all services together locally and will handle CI/CD later.

## What This Repo Is

One `docker compose up` here starts the full stack. Owns environment configuration, service wiring, and eventually deployment pipelines. No business logic lives here.

## What It Owns

- `docker-compose.yml` — full local dev stack (all services + PostgreSQL + Redis)
- `docker-compose.dev.yml` — dev overrides: hot reload, volume mounts, mock auth enabled
- `.env.example` — all required environment variables documented
- `ci/` — CI/CD pipelines (added when repos are ready for deployment)

## Services Wired

| Service | Repo | Port (local) |
|---------|------|-------------|
| cod-platform-api | ../cod-platform-api | 3000 |
| cod-platform-web | ../cod-platform-web | 3001 |
| cod-platform-ai | ../cod-platform-ai | 8000 |
| PostgreSQL | image | 5432 |
| Redis | image | 6379 |

## Session Protocol

Every agent, every coder, every session — no exceptions:

1. **Read this file first.** Do not write a single line of configuration before reading CLAUDE.md in full.
2. **Check the active plan.** Open `docs/superpowers/plans/` and find the current plan file. If no plan exists, run `/brainstorming` before touching any configuration.
3. **One task at a time.** Pick the next uncompleted task from the plan, implement it, test it, commit it, mark it done. Do not batch tasks across commits.
4. **Scope change or new service?** Stop. Run `/brainstorming` first. Update the plan. Then implement.
5. **End of session.** Commit all changes. Mark completed tasks in the plan file.

## Docs Structure

All documentation lives under `docs/`. Superpowers skills write to fixed paths — never move or rename these folders:

```
docs/
  superpowers/
    specs/    ← design specs from /brainstorming  (YYYY-MM-DD-<topic>-design.md)
    plans/    ← implementation plans from /writing-plans
```

No other doc location is valid. If a skill tries to write elsewhere, redirect it here.

## Key Rules (expand during planning)

- All service images built from their own Dockerfile — no code lives in this repo
- `.env.example` is always kept current — any new env var gets documented here immediately
- `docker-compose.dev.yml` enables mock auth by default — never commit real credentials
- Health checks on every container before services are declared ready

## Vault References

- `COD Platform/Architecture/Tech Stack.md` — service boundaries, ports, integration targets
- `Shared/Integration Contract.md` — how COD Platform and Marketing Platform connect

## Docs

Infrastructure decisions and setup guides live in `docs/` — written during planning sessions using superpowers skills before any configuration is written.
