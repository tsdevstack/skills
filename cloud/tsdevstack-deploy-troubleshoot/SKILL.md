---
name: tsdevstack-deploy-troubleshoot
description: Use when a tsdevstack cloud deploy fails or behaves wrong (first-deploy prerequisites, a cold start or timeout, an endpoint 404/401 or 5xx in the cloud, a crashing service, a service missing a secret, a minInstances error on AWS, missing traces, or GCP/AWS/Azure differences).
---

# Deploy troubleshooting

Run the right command first (see the `tsdevstack-command-model` skill), then diagnose by symptom.

## Read the logs first

Logs can hold sensitive data: ask the user before every read (environment, sources, why) and wait for a yes. `npx tsdevstack infra:logs --env <env>` (MCP: `get_logs`) lists the sources (services, workers, frontends, `kong`, `migrate-<service>`, `db-init` (the database setup `infra:deploy` runs), scheduled jobs, `bucket-<name>`, `edge`) and when each last logged. Then read one source, or `--all`, narrowed with `--since`, `--level warn`, `--kind platform` (crashes, failed health checks, deploys) or `--grep <text>`. Every line inside a request carries `trace_id`: `--all --grep <trace id>` follows one request across services. Times are UTC. Use them to debug; never copy them into files, commits, issues or messages unless the user asks. On Azure, lines arrive 1 to 10 minutes late. On GCP, a read limit (60 reads per minute) stops the command with a Logs Explorer link: narrow the window instead of retrying.

## First-deploy prerequisites

- A cloud account with credentials in `.tsdevstack/.credentials.{provider}.json`: the per-provider account-setup guide (see Reference) walks through creating both. This is the one step only a human can do.
- `cloud:init --gcp|--aws|--azure` and `cloud-secrets:push --env <env>` have run. **Never copy local secrets to the cloud**: each environment is a separate account/project, and the framework rejects credential reuse.
- `infrastructure.json` is user-authored and committed (per-env overrides).
- A **new** service needs a full `infra:deploy` (Terraform creates the runtime), not `infra:deploy-services`.
- Every Next.js app answers `GET /health` with 200 (the template's `app/health/route.ts`): the AWS load balancer, Azure App Service and Front Door check it.

## By symptom

- **404 through the gateway:** Kong only routes exact OpenAPI paths and methods. Either the gateway wasn't rebuilt after the change (`infra:generate-kong`, `infra:build-kong`, `infra:deploy-kong`; `infra:deploy-kong` alone deploys the old image), or the request doesn't match (method, trailing slash, extra segment, partner key on an endpoint without `@PartnerApi()`).
- **401/403 unexpectedly:** the auth-decorator matrix or the Kong trust boundary. See the `tsdevstack-auth-model` skill.
- **"service X is missing secret Y":** a backend reads only the shared scope and its own. Check with `cloud-secrets:get Y --service X --env <env>` (it prints the value: ask the user first). The user sets it in their own terminal, with `cloud-secrets:push` or `cloud-secrets:set Y --service X`. A frontend also needs Y in its `secrets[]` array.
- **Cold start / first request slow:** services are internal-only. On GCP and Azure, services with `minInstances: 0` scale to zero and wake on the first request (a few seconds). AWS has no scale-to-zero: every service and Kong keeps at least one task, and Kong calls services directly over Cloud Map (`http://{service}.{project}.local:8080`). SSR uses `KONG_INTERNAL_URL`; the browser uses `API_URL`.
- **Messages handled late, background work stalls:** the subscribing service has `minInstances: 0`. Its messages wait in Redis until a request starts it. Set `minInstances` to 1 or more (the default); on GCP that also keeps its CPU on between requests and needs `cpu` 1 or more.
- **AWS: "minInstances: 0 is not supported on AWS" from `infra:generate` / `infra:plan` / `infra:deploy` / `infra:status`:** `minInstances: 0` is rejected on AWS for services and Kong. Set it to 1 or more in `infrastructure.json`. Expect a higher monthly bill than with scale-to-zero.
- **AWS: every backend route errors after upgrading the CLI:** the old Kong image still targets load balancer listeners that the new `infra:deploy` removed. Rebuild and redeploy Kong (`infra:generate-kong`, `infra:build-kong`, `infra:deploy-kong`, or a full `infra:deploy`).
- **"newer Terraform providers than the ones in this environment's .terraform.lock.hcl":** the CLI pins newer providers than the lock file. Upgrading is the user's step: they run `npx tsdevstack infra:plan --env <env> --upgrade`, review the plan, commit the lock file, then deploy. Never upgrade providers without the user.
- **GCP: `infra:logs` says the account needs the Logs Viewer role, or the dashboard fails to deploy:** a project set up with an older CLI lacks `roles/logging.viewer`, `roles/monitoring.dashboardEditor` and the trace APIs. The user runs `npx tsdevstack infra:bootstrap --env <env>` once.
- **Azure: `infra:deploy` fails with an authorization error on a role assignment for the traces rule:** the deploy service principal's constrained User Access Administrator condition doesn't allow **Monitoring Metrics Publisher**. The user adds it in the portal (resource group > Access control > Role assignments > the condition). Never widen the condition beyond that role.
- **Traces missing or late:** on Cloud Run, Kong, Next.js and services with `minInstances: 0` get CPU only during requests, so their spans leave with a later request. A frontend without `instrumentation.ts` has no spans (traces start at Kong). A call made with a plain axios instance or `fetch` starts a new trace in the callee. `tracing.sampleRate` below 1 keeps only a share.
- **Azure: PostgreSQL `SkuNotAvailable` on deploy:** the subscription has no region access for that size series, even when the quota page shows the size. The user requests region access (Azure account setup, "PostgreSQL Region Access"). Microsoft grants it per series, so the v5 sizes (`GP_Standard_D2ds_v5`, `GP_Standard_D4ds_v5`, `MO_Standard_E4ds_v5`) may work when the v3 ones don't. The deploy continues from where it stopped.
- **A worker doesn't keep up with its queue:** workers scale on CPU, not queue length. Raise its `minInstances` in `infrastructure.json`.
- **Azure BullMQ `CROSSSLOT` error:** queue names need the `{bull}` hash-tag prefix.
- **Worker not updated:** workers share the service's image, so redeploy the service.

## Reference

- https://tsdevstack.dev/docs/infrastructure/architecture.md
- https://tsdevstack.dev/docs/features/cloud-logs.md
- https://tsdevstack.dev/docs/features/traces-and-dashboards.md
- https://tsdevstack.dev/docs/infrastructure/environments.md
- https://tsdevstack.dev/docs/secrets/cloud-secrets.md
- https://tsdevstack.dev/docs/features/scheduled-jobs.md
- Account setup per provider: https://tsdevstack.dev/docs/infrastructure/providers/gcp/account-setup.md · https://tsdevstack.dev/docs/infrastructure/providers/aws/account-setup.md · https://tsdevstack.dev/docs/infrastructure/providers/azure/account-setup.md
- Deploys from CI: https://tsdevstack.dev/docs/infrastructure/cicd-setup.md
