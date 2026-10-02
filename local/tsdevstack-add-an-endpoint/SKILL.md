---
name: tsdevstack-add-an-endpoint
description: Use when adding, exposing, or changing a NestJS API endpoint in a tsdevstack app, or when an endpoint returns 404 through the gateway. Covers the decorator → OpenAPI → Kong → client pipeline and choosing the auth type.
---

# Add an API endpoint (tsdevstack)

An endpoint is exposed through a generated pipeline, not by editing gateway
config. Decorators on the controller drive everything downstream. Kong only
routes what is in the OpenAPI spec, exactly: the path and its declared methods.
Skip a step and the route 404s at the gateway.

## Pipeline (in order)

1. **Add the controller route with decorators.** Every route needs OpenAPI
   decorators (`@ApiOperation`, `@ApiResponse`) plus the right auth decorator
   (below). A route excluded from OpenAPI (`@ApiExcludeEndpoint()`) gets no
   gateway route.
2. **Regenerate the OpenAPI spec:** `npm run docs:generate -w <service>`.
3. **Regenerate the gateway config:** `npx tsdevstack generate-kong` (or
   `npx tsdevstack sync`, which does steps 2 and 3 and restarts the stack).
   Until you do, Kong answers 404 for the new path or method.
4. **Regenerate clients** if the frontend or another service calls this
   endpoint: `npx tsdevstack generate-client`.
5. **Deploy (cloud):** `npx tsdevstack infra:deploy-services --env <env>`, then
   the gateway: `npx tsdevstack infra:generate-kong --env <env>`,
   `infra:build-kong --env <env>`, `infra:deploy-kong --env <env>` (commit the
   regenerated `infrastructure/kong/<env>/kong.yml`). See the
   `tsdevstack-command-model` skill for what each command already includes.

## Choose the auth type (the decorator you pick)

- `@Public()`: no auth. Public route.
- `@ApiBearerAuth()`: JWT / OIDC. Logged-in users. Add `@Roles('ADMIN')` (or a
  custom role) to restrict it further.
- `@PartnerApi()`: partner API key. Kong serves this one operation at `/api` +
  its path, checks the `x-api-key` (and its limits) against Redis, and removes
  `/api` before it reaches your service. Partner keys cannot reach endpoints
  without it. Keys are issued at runtime through the auth service's admin API,
  not configured; see https://tsdevstack.dev/docs/authentication/api-keys.md.
- `@ApiBearerAuth()` + `@PartnerApi()`: dual access (JWT and API key) on the
  same handler.

Internal service-to-service calls don't need a decorator: they bypass Kong and
authenticate with the service `API_KEY`.

## Gotchas

- **Never edit generated `infrastructure/kong/**/kong.yml`**: it's regenerated.
  For custom gateway behavior use `kong.user.yml` (merge) or `kong.custom.yml`
  (full override).
- A 404 through the gateway (`no Route matched`) usually means the Kong config
  wasn't regenerated (and, in the cloud, rebuilt and redeployed) after the
  change, or the request doesn't match exactly: wrong method, trailing slash,
  extra path segment, or a partner key on an endpoint without `@PartnerApi()`.
- A 401 on a route without `@ApiBearerAuth()` means `@Public()` is missing (see
  the `tsdevstack-auth-model` skill).

## Reference

- https://tsdevstack.dev/docs/building-apis/openapi-decorators.md
- https://tsdevstack.dev/docs/building-apis/gateway-routing.md
- https://tsdevstack.dev/docs/building-apis/dto-generation.md
