---
name: tsdevstack-run-locally
description: Use when starting a tsdevstack project locally, when a container or service won't come up, when you hit a port conflict, or when a route returns 502/404 in local dev.
---

# Run a tsdevstack project locally

Bring it up with `npx tsdevstack sync` (regenerates docker-compose, Kong, secrets) then `npm run dev`. Kong at `:8000` is the entry point; each service also runs on its own port (`:3001`…) for direct debugging. **Node 22+ is required** — older Node causes cascading install failures.

## Won't start / unhealthy

1. Is Docker running?
2. **Port conflict** — tsdevstack binds `8000` (Kong), `5432`+ (per-service Postgres), `6379` (Redis), plus tooling (pgAdmin `5050`, Redis Commander `8081`, Jaeger `16686`, MinIO `9000`/`9001`). If a host process already owns one, the container fails to start.
   - Find it: `lsof -i :5432` (swap in the conflicting port), then stop that process.
   - Can't stop it? Override the port in `docker-compose.user.yml` (auto-included) — **never edit the generated `docker-compose.yml`.**
3. First-run DB race (`P1010`): re-run `npx tsdevstack sync` — it waits on container healthchecks.

## 502 / 404 through the gateway

Almost always stale generated config: re-run `npx tsdevstack sync` and restart. A new or changed endpoint also needs its decorators (see the `tsdevstack-add-an-endpoint` skill).

## Local quirks

- Backends read secrets via `SecretsService` (generated into `.secrets.local.json`), not `.env`.
- Email goes to the **console** locally — watch the terminal for verification/reset links; nothing is actually sent.
- `KONG_SERVICE_HOST` = `host.docker.internal` on Mac/Windows, `172.17.0.1` on Linux; the gateway container is `tsdevstack-gateway-1`.
- The package is `@tsdevstack/cli` (bins `tsdevstack` / `tsds`). A 404 installing `tsdevstack@*` means a broken install.

## Reference

- https://tsdevstack.dev/docs/local-development/running-locally.md
- https://tsdevstack.dev/docs/getting-started/quick-start.md
- https://tsdevstack.dev/docs/local-development/debugging.md
