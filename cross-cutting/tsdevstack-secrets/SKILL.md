---
name: tsdevstack-secrets
description: Use when adding, reading, or debugging a secret or config value in a tsdevstack service — where it goes, how to read it in code, or fixing a missing/undefined secret. Backends read via SecretsService, never process.env.
---

# Secrets

Backends read config via `SecretsService`, never `process.env`. Values live in three files locally, and in the cloud provider's secret store when deployed.

## Read a secret (backend)

Inject `SecretsService` (from `@tsdevstack/nest-common`) and `await this.secrets.get('KEY')` (returns a string). Never read secrets from `process.env`.

## The three files (local)

- `.secrets.tsdevstack.json` — framework-generated (DB creds, JWT keys, service API keys, `KONG_TRUST_TOKEN`). **Don't edit.**
- `.secrets.user.json` — **yours.** Add your own keys here (third-party API keys, TTL overrides).
- `.secrets.local.json` — the merged result the app actually reads. Generated, gitignored. **Don't edit.**

## Add a secret

1. Add the key to `.secrets.user.json`.
2. Scope it to the services that need it (their per-service `secrets: []` list) — a backend only receives the secrets it's scoped for.
3. Run `npx tsdevstack generate-secrets` (or `sync`) to rebuild `.secrets.local.json`.

## Cloud

`npx tsdevstack cloud-secrets:push --env <env>` pushes user + framework secrets; set one with `cloud-secrets:set <KEY> --env <env>`. **Never copy local secrets to the cloud** — each environment is a separate account/project.

## Missing / undefined secret

- Confirm the key is in `.secrets.user.json` **and** scoped to the service, then re-run `generate-secrets` / `sync`.
- In the cloud: confirm `cloud-secrets:push` ran and the service is in scope (shared scope is looked up before service scope).

## Reference

- https://tsdevstack.dev/docs/secrets/how-secrets-work.md
- https://tsdevstack.dev/docs/secrets/user-vs-framework.md
- https://tsdevstack.dev/docs/secrets/local-secrets.md
- https://tsdevstack.dev/docs/secrets/cloud-secrets.md
