---
name: tsdevstack-service-and-worker-lifecycle
description: Use when adding or removing a service, or adding or removing a worker, in a tsdevstack project. Always through the CLI — never hand-create or delete service folders or config.
---

# Add / remove services and workers

Scaffold and remove **only through the CLI.** Never hand-create a service folder or edit `.tsdevstack/config.json` — the CLI derives Kong routes, docker-compose, secrets, and clients from it, and a manual change desyncs all of them.

## Add a service

- `npx tsdevstack add-service --type nestjs|nextjs|spa`. NestJS names must be lowercase and end `-service`; reserved names (`gateway`, `kong`, `redis`, `postgres`, …) are rejected.
- Then: `npm install` → `npx tsdevstack sync`.
- Routes are served at `/{short-name}/` (the name minus `-service`).

## Remove a service

- `npx tsdevstack remove-service <name>` → `npx tsdevstack sync`.
- Renaming is manual only (cloud DBs can't be renamed) — remove and re-add if you must.

## Workers

- **Inline worker** — runs in the same container; `worker.ts` / `worker.module.ts` bootstrapped via `startWorker(WorkerModule)`. Job code patterns → the `tsdevstack-nest-common` skill.
- **Detached worker** — separate container (minimum 1 instance), entrypoint `dist/worker.js`. Register with `npx tsdevstack register-detached-worker`, remove with `npx tsdevstack unregister-detached-worker` (config only — no scaffolding).
- A `worker.ts` file alone does nothing until it's registered.

## After any change

`sync` regenerates local config. In the cloud, a **new** service needs a full `infra:deploy` (Terraform creates the runtime) — see the `tsdevstack-command-model` skill.

## Reference

- https://tsdevstack.dev/docs/local-development/adding-apps.md
- https://tsdevstack.dev/docs/introduction/supported-apps.md
