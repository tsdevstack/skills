---
name: tsdevstack-db-changes
description: Use when adding or changing a database model, field, or table in a tsdevstack service, creating or applying Prisma migrations, or debugging a migration or schema-drift error. Migrations are created locally with prisma migrate dev; sync and cloud deploys only apply committed ones.
---

# Database changes (Prisma migrations)

Each NestJS service with a database owns `apps/<service>/prisma/schema.prisma` and its own Postgres. Migrations are created in exactly one place — locally, by you. Everything downstream only applies what you committed.

## Change a schema

1. Edit `apps/<service>/prisma/schema.prisma`.
2. From the **service directory**: `npx prisma migrate dev --name <change>` — creates the migration, applies it to the local DB, regenerates the client. `DATABASE_URL` comes from the generated `.env` in the service dir; never set it by hand.
3. Commit the new `prisma/migrations/` folder together with the code that uses it.

## Who runs what

- `prisma migrate dev` (you, locally) — the **only** step that creates migrations.
- `npx tsdevstack sync` — applies committed migrations and regenerates clients for every DB service (e.g. after pulling changes). Never creates them.
- Cloud deploys (`infra:deploy` / `infra:deploy-services`) — run `prisma migrate deploy` per DB service in a short-lived cloud job before the new revision goes live. **Do not run migrations by hand after a deploy** — they already ran.
- `infra:plan-db-migrate` / `infra:run-db-migrate --service <name> --env <env>` — preview / apply in the cloud without a full deploy (recovery or CI only).

## Never

- Edit or delete an applied migration — create a new migration that makes the change.
- Point `prisma migrate dev` at a cloud database — the cloud only ever applies (`migrate deploy`).
- Edit the generated `.env` in a service directory — it is overwritten by `generate-secrets` / `sync`.

## Reference

- https://tsdevstack.dev/docs/local-development/database-migrations.md
