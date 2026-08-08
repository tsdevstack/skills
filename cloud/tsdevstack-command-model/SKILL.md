---
name: tsdevstack-command-model
description: Use before running tsdevstack deploy or generate commands to know what each one does and which steps it already includes, so you don't run redundant or conflicting commands, such as choosing infra:deploy vs infra:deploy-services or whether to also run infra:deploy-kong and infra:deploy-lb after a full deploy.
---

# tsdevstack command model

Many commands are composed — the big ones already run the smaller ones. Running
a smaller one again after a full deploy is redundant and can cause churn. Match
the command to what actually changed.

## What the big command includes

`npx tsdevstack infra:deploy --env <env>` runs the full chain: provision
infrastructure (Terraform) → sync post-provision secrets → build images → push →
deploy services → deploy gateway (Kong) → deploy load balancer.

**After a full `infra:deploy`, do NOT separately run `infra:deploy-kong` or
`infra:deploy-lb`** — they already ran.

## Match the command to the change

| Changed | Run | Not needed |
|---|---|---|
| Infra (DB tier, buckets, new service runtime) | `infra:deploy` | the rest — it's included |
| Service code only | `infra:deploy-services` (or `infra:deploy-service <name>`) | `infra:deploy` (slower, unneeded) |
| Endpoint decorators / routes | `generate-kong` then `infra:deploy-kong` | full deploy |
| Domains / SSL | `infra:deploy-lb` | full deploy |
| Scheduled jobs | `infra:deploy-schedulers` | full deploy |

## Local vs cloud

- `sync` regenerates **local** config (docker-compose, kong, secrets) after
  adding services or changing config. It does not touch the cloud.
- Always preview infra changes with `infra:plan --env <env>` before
  `infra:deploy`.

## Gotchas

- These exist as both CLI (`infra:deploy`) and MCP tools (`infra_deploy`) —
  same behavior.
- For the authoritative, current list and flags, defer to the docs and
  `npx tsdevstack <command> --help`. This skill is the mental model, not the spec.

## Reference

- https://tsdevstack.dev/docs/reference/cli-commands.md
- https://tsdevstack.dev/docs/infrastructure/environments.md
