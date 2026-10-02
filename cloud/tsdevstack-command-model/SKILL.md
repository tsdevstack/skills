---
name: tsdevstack-command-model
description: Use before running tsdevstack deploy or generate commands to know what each one does and which steps it already includes, so you don't run redundant or conflicting commands, such as choosing infra:deploy vs infra:deploy-services or whether to also run infra:deploy-kong and infra:deploy-lb after a full deploy.
---

# tsdevstack command model

Many commands are composed: the big ones already run the smaller ones. Running
a smaller one again after a full deploy is redundant and can cause churn. Match
the command to what actually changed.

## What the big command includes

`npx tsdevstack infra:deploy --env <env>` runs the full chain: provision
infrastructure (Terraform) → sync post-provision secrets → build images → push →
deploy services → gateway (Kong: generate, build, deploy) → deploy load balancer.

**After a full `infra:deploy`, do NOT separately run `infra:deploy-kong` or
`infra:deploy-lb`**: they already ran.

## The gateway is three steps

Kong changes reach the cloud only through a rebuilt image:

1. `infra:generate-kong --env <env>`: writes `infrastructure/kong/<env>/kong.yml` from the OpenAPI specs and `kong.user.yml` (commit it).
2. `infra:build-kong --env <env>`: resolves secrets into the config, builds and pushes the image, including your `kong-plugins/` folder.
3. `infra:deploy-kong --env <env>`: deploys that image.

`infra:deploy-kong` alone deploys an already built image; it does not pick up new routes or plugins.

## Match the command to the change

| Changed | Run | Not needed |
|---|---|---|
| Infra (DB tier, buckets, new service runtime) | `infra:deploy` | the rest: it's included |
| Service code only | `infra:deploy-services` (or `infra:deploy-service <name>`) | `infra:deploy` (slower, unneeded) |
| Endpoint decorators / routes, `kong.user.yml`, `kong-plugins/` | `infra:generate-kong`, `infra:build-kong`, `infra:deploy-kong` | full deploy |
| CLI upgrade (framework Kong plugins change) | the same three gateway steps | full deploy |
| Domains / SSL | `infra:deploy-lb` | full deploy |
| Scheduled jobs | `infra:deploy-schedulers` | full deploy |
| `framework.apiKeys.ipLimitPerMinute`, global `rate-limiting` (API key defaults) | the three gateway steps | full deploy |
| API keys (create, limits, revoke, rotate) | nothing: admin API at runtime | any deploy |

## Auth template: the API key usage job

`infra:generate` warns per environment when `scheduledJobs` lacks
`sync-api-key-usage` (every 5 minutes, `/auth/jobs/sync-api-key-usage` on the
auth-service) and prints the entry. Add it to `.tsdevstack/infrastructure.json`
and run `infra:deploy-schedulers`. Without it, API key usage is never saved and
a lost Redis key index is only rebuilt when the auth-service restarts.

## Local vs cloud

- `sync` regenerates **local** config (docker-compose, kong, secrets, the Kong
  image files in `infrastructure/kong/`) and rebuilds the local gateway image.
  Run it on a fresh clone before `docker compose up`. It does not touch the
  cloud.
- Always preview infra changes with `infra:plan --env <env>` before
  `infra:deploy`.

## Gotchas

- These exist as both CLI (`infra:deploy`) and MCP tools (`infra_deploy`),
  same behavior.
- For the authoritative, current list and flags, defer to the docs and
  `npx tsdevstack <command> --help`. This skill is the mental model, not the spec.

## Reference

- https://tsdevstack.dev/docs/reference/cli-commands.md
- https://tsdevstack.dev/docs/infrastructure/environments.md
