---
name: tsdevstack-secrets
description: Use when adding, reading, or debugging a secret or config value in a tsdevstack service (where it goes, how to read it in code, fixing a missing/undefined secret, or setting ADMIN_EMAILS for the first admins). Backends read via SecretsService, never process.env.
---

# Secrets

Backends read config via `SecretsService`, never `process.env`. Values live in three files locally, and in the cloud provider's secret store when deployed.

## Read a secret (backend)

Inject `SecretsService` (from `@tsdevstack/nest-common`) and `await this.secrets.get('KEY')` (returns a string). Never read secrets from `process.env`.

## The three files (local)

- `.secrets.tsdevstack.json`: framework-generated (DB creds, JWT keys, service API keys, `KONG_TRUST_TOKEN`). **Don't edit.**
- `.secrets.user.json`: **yours.** Add your own keys here (third-party API keys, TTL overrides, `ADMIN_EMAILS`).
- `.secrets.local.json`: the merged result the app actually reads. Generated, gitignored. **Don't edit.**

## Add a secret

1. Add the key to `.secrets.user.json`.
2. Scope it to the services that need it (their per-service `secrets: []` list): a backend only receives the secrets it's scoped for.
3. Run `npx tsdevstack generate-secrets` (or `sync`) to rebuild `.secrets.local.json`.

## First admins (`ADMIN_EMAILS`, auth template)

`ADMIN_EMAILS` is a comma-separated list of emails. A confirmed user on the list becomes `ADMIN` at their next login. Locally it's in `.secrets.user.json` (created empty, already scoped to the auth service); run `generate-secrets` / `sync` and restart the auth service after editing it. In the cloud it lives in the **auth service's scope**, not shared: `npx tsdevstack cloud-secrets:set ADMIN_EMAILS --service auth-service --env <env>`. `cloud-secrets:push` offers it only when your local value is set, and prints that command otherwise.

## Partner API keys are not secrets

Keys for partners (`@PartnerApi()` routes) are created at runtime through the auth service's admin API and stored hashed in Postgres and Redis. Don't add them to `.secrets.user.json` or the cloud secret manager, and don't reference them from `kong.user.yml`. Old static partner key values can be removed (`cloud-secrets:remove <KEY> --env <env>`). See https://tsdevstack.dev/docs/authentication/api-keys.md.

## Cloud

`npx tsdevstack cloud-secrets:push --env <env>` pushes user + framework secrets; set one with `cloud-secrets:set <KEY> --env <env>` (add `--service <name>` for a service-scoped secret). **Never copy local secrets to the cloud**: each environment is a separate account/project.

## Missing / undefined secret

- Confirm the key is in `.secrets.user.json` **and** scoped to the service, then re-run `generate-secrets` / `sync`.
- In the cloud: confirm `cloud-secrets:push` ran and the service is in scope (shared scope is looked up before service scope).

## Reference

- https://tsdevstack.dev/docs/secrets/how-secrets-work.md
- https://tsdevstack.dev/docs/secrets/user-vs-framework.md
- https://tsdevstack.dev/docs/secrets/local-secrets.md
- https://tsdevstack.dev/docs/secrets/cloud-secrets.md
