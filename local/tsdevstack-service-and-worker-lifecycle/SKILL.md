---
name: tsdevstack-service-and-worker-lifecycle
description: Use when adding or removing a service, or adding or removing a worker, in a tsdevstack project. Always through the CLI — never hand-create or delete service folders or config.
---

# Add / remove services and workers

Scaffold and remove **only through the CLI.** Never hand-create a service folder or edit `.tsdevstack/config.json` — the CLI derives Kong routes, docker-compose, secrets, and clients from it, and a manual change desyncs all of them.

## Add a service

- `npx tsdevstack add-service --type nestjs|nextjs|spa`. NestJS names must be lowercase and end `-service`; reserved names (`gateway`, `kong`, `redis`, `postgres`, `shared`, `secrets`, `kv`, …) and names starting with `shared-` are rejected.
- Then: `npm install` → `npx tsdevstack sync`.
- Routes are served at `/{short-name}/` (the name minus `-service`).

## Remove a service

- `npx tsdevstack remove-service <name>` → `npx tsdevstack sync`.
- A service that subscribes to messaging topics: first remove its subscriptions and deploy it, wait ~15 minutes, then remove it (see the `tsdevstack-messaging` skill).
- Renaming is manual only (cloud DBs can't be renamed) — remove and re-add if you must.

## Workers

- **Queue processors run only in workers**, never in a service's `AppModule` (a service can scale to zero, and on GCP it then gets CPU only during requests). The service registers the queue to add jobs; the worker registers it and holds the processor.
- **Worker files** in the base service: `src/worker.ts` / `src/worker.module.ts` bootstrapped via `startWorker(WorkerModule)`, `tsconfig.worker.json` (local build into `node_modules/.cache/worker-dev`), and the scripts `start-worker` (`node dist/worker.js`) and `start-worker:dev` (`nest start --entryFile worker --watch --path tsconfig.worker.json`). `npm run dev` starts every worker. Job code patterns → the `tsdevstack-nest-common` skill. The auth template ships with `auth-worker`.
- **In the cloud** — separate container (minimum 1 instance), entrypoint `dist/worker.js`. It scales on CPU between `minInstances` and `maxInstances` (70% on AWS and Azure, 60% on GCP), not on queue length: a processor whose jobs wait on the network (emails) gets `concurrency` (plus a `limiter` for a rate-limited API, like the auth template's email processor); CPU-heavy jobs scale by adding workers. Register with `npx tsdevstack register-detached-worker`: it also creates whichever worker files and scripts are missing (never overwriting one); the processors are yours. Remove with `npx tsdevstack unregister-detached-worker` (config only; it never deletes code).
- A `worker.ts` file alone does nothing in the cloud until it's registered.

## After any change

`sync` regenerates local config. In the cloud, a **new** service needs a full `infra:deploy` (Terraform creates the runtime) — see the `tsdevstack-command-model` skill.

## Reference

- https://tsdevstack.dev/docs/local-development/adding-apps.md
- https://tsdevstack.dev/docs/introduction/supported-apps.md
