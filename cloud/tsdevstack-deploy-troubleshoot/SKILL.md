---
name: tsdevstack-deploy-troubleshoot
description: Use when a tsdevstack cloud deploy fails or behaves wrong (first-deploy prerequisites, a cold start or timeout, an endpoint 404/401 in the cloud, a service missing a secret, a minInstances error on AWS, or GCP/AWS/Azure differences).
---

# Deploy troubleshooting

Run the right command first (see the `tsdevstack-command-model` skill), then diagnose by symptom.

## First-deploy prerequisites

- A cloud account with credentials in `.tsdevstack/.credentials.{provider}.json`: the per-provider account-setup guide (see Reference) walks through creating both. This is the one step only a human can do.
- `cloud:init --gcp|--aws|--azure` and `cloud-secrets:push --env <env>` have run. **Never copy local secrets to the cloud**: each environment is a separate account/project, and the framework rejects credential reuse.
- `infrastructure.json` is user-authored and committed (per-env overrides).
- A **new** service needs a full `infra:deploy` (Terraform creates the runtime), not `infra:deploy-services`.

## By symptom

- **404 through the gateway:** Kong only routes exact OpenAPI paths and methods. Either the gateway wasn't rebuilt after the change (`infra:generate-kong`, `infra:build-kong`, `infra:deploy-kong`; `infra:deploy-kong` alone deploys the old image), or the request doesn't match (method, trailing slash, extra segment, partner key on an endpoint without `@PartnerApi()`).
- **401/403 unexpectedly:** the auth-decorator matrix or the Kong trust boundary. See the `tsdevstack-auth-model` skill.
- **"service X is missing secret Y":** secret scoping: check the service's `secrets[]`, that `cloud-secrets:push` ran, and remember shared scope is looked up first.
- **Cold start / first request slow:** services are internal-only. On GCP and Azure, services with `minInstances: 0` scale to zero and wake on the first request (a few seconds). AWS has no scale-to-zero: every service and Kong keeps at least one task, and Kong calls services directly over Cloud Map (`http://{service}.{project}.local:8080`). SSR uses `KONG_INTERNAL_URL`; the browser uses `API_URL`.
- **AWS: "minInstances: 0 is not supported on AWS" from `infra:generate` / `infra:plan` / `infra:deploy` / `infra:status`:** `minInstances: 0` is rejected on AWS for services and Kong. Set it to 1 or more in `infrastructure.json`. Expect a higher monthly bill than with scale-to-zero.
- **AWS: every backend route errors after upgrading the CLI:** the old Kong image still targets load balancer listeners that the new `infra:deploy` removed. Rebuild and redeploy Kong (`infra:generate-kong`, `infra:build-kong`, `infra:deploy-kong`, or a full `infra:deploy`).
- **Azure BullMQ `CROSSSLOT` error:** queue names need the `{bull}` hash-tag prefix.
- **Worker not updated:** workers share the service's image, so redeploy the service.

## Reference

- https://tsdevstack.dev/docs/infrastructure/architecture.md
- https://tsdevstack.dev/docs/infrastructure/environments.md
- https://tsdevstack.dev/docs/secrets/cloud-secrets.md
- https://tsdevstack.dev/docs/features/scheduled-jobs.md
- Account setup per provider: https://tsdevstack.dev/docs/infrastructure/providers/gcp/account-setup.md · https://tsdevstack.dev/docs/infrastructure/providers/aws/account-setup.md · https://tsdevstack.dev/docs/infrastructure/providers/azure/account-setup.md
- Deploys from CI: https://tsdevstack.dev/docs/infrastructure/cicd-setup.md
