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
