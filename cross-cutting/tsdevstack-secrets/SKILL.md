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

A backend reads two scopes: **shared** and **its own**. Its own value wins when a key is in both. In the cloud, each service's identity can read only these two. Locally, nothing enforces it.

1. Add the value to `.secrets.user.json`. Where it goes decides who reads it:
   - In a service's section (`"offers-service": { "STRIPE_KEY": "sk_test_..." }`): that service's own scope. Use this for a secret one service needs.
   - In the top-level `"secrets"` object: shared, readable by every backend. Only for values every service needs.
2. Run `npx tsdevstack generate-secrets` (or `sync`) to rebuild `.secrets.local.json`. Backends pick it up within a minute, no restart.

Frontends (Next.js, SPA) are different: at deploy they get the shared keys listed in their `secrets: []` array as environment variables. For backends that array changes nothing.

## First admins (`ADMIN_EMAILS`, auth template)

`ADMIN_EMAILS` is a comma-separated list of emails. A confirmed user on the list becomes `ADMIN` at their next login.

- Locally: edit it in `.secrets.user.json` (created empty), then run `generate-secrets` or `sync`. It lands in the auth service's scope, and the auth service reads it within a minute.
- In the cloud: it lives in the **auth service's scope**, not shared. Set it with `npx tsdevstack cloud-secrets:set ADMIN_EMAILS --service auth-service --env <env>`. `cloud-secrets:push` offers it only when your local value is set, and prints that command otherwise.

## Partner API keys are not secrets

Keys for partners (`@PartnerApi()` routes) are created at runtime through the auth service's admin API and stored hashed in Postgres and Redis. Don't add them to `.secrets.user.json` or the cloud secret manager, and don't reference them from `kong.user.yml`. Old static partner key values can be removed (`cloud-secrets:remove <KEY> --env <env>`). See https://tsdevstack.dev/docs/authentication/api-keys.md.

## Cloud

`npx tsdevstack cloud-secrets:push --env <env>` pushes the framework secrets. It asks for each environment's value of your secrets and puts them in the same scopes as locally. The user runs these commands in their own terminal: an agent gives the command and never asks for a secret value. Set one with `cloud-secrets:set <KEY> --env <env>`. Add `--service <name>` for a service's scope, and `--stdin` to pipe the value in. **Never copy local secrets to the cloud**: each environment is a separate account/project.

An environment deployed before 0.11 keeps the auth secrets in shared. On Azure, it also keeps service secrets in the shared vault. Upgrade it in this order: `npx tsdevstack cloud-secrets:migrate --env <env>`, then `infra:deploy`, then `cloud-secrets:cleanup --env <env>` to delete the old copies.

## Missing / undefined secret

- Backend: the key must be in the top-level `"secrets"` object or in **this** service's section of `.secrets.user.json`. A value in another service's section isn't readable. Then re-run `generate-secrets` or `sync`.
- Frontend: the key must be in its `secrets: []` array, and the frontend redeployed.
- In the cloud: `cloud-secrets:get <KEY> --service <name> --env <env>` shows whether it's in the service's scope. It prints the value, so ask the user before you run it. Without `--service`, it checks the key's default scope. A service never reads another service's scope.

## Reference

- https://tsdevstack.dev/docs/secrets/how-secrets-work.md
- https://tsdevstack.dev/docs/secrets/user-vs-framework.md
- https://tsdevstack.dev/docs/secrets/local-secrets.md
- https://tsdevstack.dev/docs/secrets/cloud-secrets.md
