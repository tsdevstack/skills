---
name: tsdevstack-deploy-troubleshoot
description: Use when a tsdevstack cloud deploy fails or behaves wrong — first-deploy prerequisites, a cold start or timeout, an endpoint 404/401 in the cloud, a service missing a secret, or GCP/AWS/Azure differences.
---

# Deploy troubleshooting

Run the right command first (see the `tsdevstack-command-model` skill), then diagnose by symptom.

## First-deploy prerequisites

- A cloud account with credentials in `.tsdevstack/.credentials.{provider}.json` — the per-provider account-setup guide (see Reference) walks through creating both. This is the one step only a human can do.
- `cloud:init --gcp|--aws|--azure` and `cloud-secrets:push --env <env>` have run. **Never copy local secrets to the cloud** — each environment is a separate account/project, and the framework rejects credential reuse.
- `infrastructure.json` is user-authored and committed (per-env overrides).
- A **new** service needs a full `infra:deploy` (Terraform creates the runtime), not `infra:deploy-services`.

## By symptom

- **404 through the gateway:** missing route decorators, or `generate-kong` / `infra:deploy-kong` not run after the change.
- **401/403 unexpectedly:** the auth-decorator matrix or the Kong trust boundary — see the `tsdevstack-auth-model` skill.
- **"service X is missing secret Y":** secret scoping — check the service's `secrets[]`, that `cloud-secrets:push` ran, and remember shared scope is looked up first.
- **Cold start / first request slow or 502:** services are internal-only and scale to zero. On AWS, Kong must be `minInstances: 1` and wakes the others; on GCP/Azure services wake on first request. SSR uses `KONG_INTERNAL_URL`; the browser uses `API_URL`.
- **Azure BullMQ `CROSSSLOT` error:** queue names need the `{bull}` hash-tag prefix.
- **Worker not updated:** workers share the service's image — redeploy the service.

## Reference

- https://tsdevstack.dev/docs/infrastructure/architecture.md
- https://tsdevstack.dev/docs/infrastructure/environments.md
- https://tsdevstack.dev/docs/secrets/cloud-secrets.md
- https://tsdevstack.dev/docs/features/scheduled-jobs.md
- Account setup per provider: https://tsdevstack.dev/docs/infrastructure/providers/gcp/account-setup.md · https://tsdevstack.dev/docs/infrastructure/providers/aws/account-setup.md · https://tsdevstack.dev/docs/infrastructure/providers/azure/account-setup.md
- Deploys from CI: https://tsdevstack.dev/docs/infrastructure/cicd-setup.md
