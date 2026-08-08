---
name: tsdevstack-add-an-endpoint
description: Use when adding, exposing, or changing a NestJS API endpoint in a tsdevstack app, or when an endpoint returns 404 through the gateway. Covers the decorator → OpenAPI → Kong → client pipeline and choosing the auth type.
---

# Add an API endpoint (tsdevstack)

An endpoint is exposed through a generated pipeline, not by editing gateway
config. Decorators on the controller drive everything downstream. Skip a step
and the route 404s in production.

## Pipeline (in order)

1. **Add the controller route with decorators.** Every route needs OpenAPI
   decorators (`@ApiOperation`, `@ApiResponse`) plus one auth decorator (below).
   No decorator → no gateway route.
2. **Regenerate the OpenAPI spec:** `npm run docs:generate -w <service>`.
3. **Regenerate the gateway config:** `npx tsdevstack generate-kong`.
4. **Regenerate clients** if the frontend or another service calls this
   endpoint: `npx tsdevstack generate-client`.
5. **Deploy (cloud):** `npx tsdevstack infra:deploy-services --env <env>` then
   `npx tsdevstack infra:deploy-kong --env <env>`. See the `tsdevstack-command-model` skill
   for what each command already includes.

## Choose the auth type (the decorator you pick)

- `@Public()` — no auth. Public route.
- `@ApiBearerAuth()` — JWT / OIDC. Logged-in users.
- `@PartnerApi()` — API key. Service-to-service or partner access, served under
  `/api/...` with the prefix stripped.
- `@ApiBearerAuth()` + `@PartnerApi()` — dual access (JWT and API key) on the
  same handler.

## Gotchas

- **Never edit generated `infrastructure/kong/**/kong.yml`** — it's regenerated.
  For custom gateway behavior use `kong.user.yml` (merge) or `kong.custom.yml`
  (full override).
- A 404 through the gateway almost always means a missing auth decorator, or
  that `generate-kong` / `infra:deploy-kong` wasn't run after the change.

## Reference

- https://tsdevstack.dev/docs/building-apis/openapi-decorators.md
- https://tsdevstack.dev/docs/building-apis/gateway-routing.md
- https://tsdevstack.dev/docs/building-apis/dto-generation.md
