---
name: tsdevstack-auth-model
description: Use when adding or changing authentication on a tsdevstack endpoint, choosing between @Public / @ApiBearerAuth / @PartnerApi, reading the authenticated user (req.user), wiring service-to-service calls, or debugging an unexpected 401/403 through the gateway.
---

# tsdevstack auth model

Auth is **two independent layers**. Getting the combination wrong is the single most common tsdevstack mistake. Kong validates the JWT at the edge; your backend never validates a JWT itself.

## The two layers

1. **Kong (gateway)** — controlled by `@ApiBearerAuth()`. Present ⇒ Kong's OIDC plugin requires and validates the JWT (RS256, via the auth service's JWKS) before the request reaches your service. Absent ⇒ Kong forwards the route with no JWT check.
2. **AuthGuard (backend, global)** — controlled by `@Public()`. It runs on every route (registered as `APP_GUARD` from `@tsdevstack/nest-common`; the class is `AuthGuard`). Absent ⇒ it requires an authenticated user or a valid service API key. Present ⇒ it skips user auth. Either way it verifies the Kong trust header.

## The decorator matrix (get this right)

| Intent | `@ApiBearerAuth()` | `@Public()` |
|---|---|---|
| Logged-in users only | yes | no |
| Truly public | no | yes |
| **Bug — 401 in prod** | no | no |
| (rarely useful) | yes | yes |

**Omitting both is the classic bug:** Kong lets the request through (no JWT required), then the global `AuthGuard` rejects it (`No authentication provided`) → 401. A public route needs **both** `@Public()` *and* no `@ApiBearerAuth()`.

Decorators come from `@tsdevstack/nest-common` (`@Public`, `@PartnerApi`, `@Partner`); `@ApiBearerAuth` is the standard Swagger decorator. `generate-kong` reads them from the OpenAPI spec — an **undecorated route gets no gateway route at all**.

## Reading the user

Type the request as `AuthenticatedRequest` (from nest-common). On JWT routes:

- `req.user.id` — the user id (the JWT `sub` claim). **Use `.id`, never `.sub`.**
- `req.user.<claim>` — any other claim (email, roles, tenantId, …), typed dynamically, parsed to its real type.

Kong forwards the claims (via `X-Userinfo`, with legacy `X-JWT-Claim-*` fallback); your backend does **not** decode or validate the JWT — just read `req.user`.

## Partner / API-key access

- `@PartnerApi()` exposes the route under `/api/{prefix}/...` with Kong key-auth (`x-api-key`). Combine with `@ApiBearerAuth()` for dual access (JWT at `/{prefix}`, key at `/api/{prefix}`).
- `@Partner()` param decorator gives the consumer name. For key/service requests `req.service` is set (instead of `req.user`).

## Service-to-service

Each backend receives `{SERVICE}_SERVICE_API_KEY` + `{SERVICE}_URL` for its peers, plus its own `API_KEY`. Use `BaseServiceClient` + `filterForwardHeaders()` from nest-common. Direct calls send `x-api-key` + `x-service-name` and bypass Kong; `AuthGuard` validates the key against the callee's `API_KEY`.

## Trust boundary

Kong stamps `X-Kong-Trust` (= the `KONG_TRUST_TOKEN` secret) on every proxied request; `AuthGuard` timing-safe-compares it, so a request that skipped Kong is rejected — except `/.well-known/*`, `/health`, and `/metrics` (infra probes).

## Templates

`framework.template` in `.tsdevstack/config.json`:

- `fullstack-auth` — auth service + Next.js BFF.
- `auth` — auth service only.
- `null` — no auth service; bring external OIDC via the `OIDC_DISCOVERY_URL` secret.

JWT signing keys are generated only for `auth` / `fullstack-auth`.

## Never

- Validate or decode JWTs in your backend — Kong does it.
- Read auth values from `process.env` — `AuthGuard` and `SecretsService` handle them.
- Assume a route is protected without `@ApiBearerAuth()` — Kong is what enforces the JWT.

## Reference

- https://tsdevstack.dev/docs/authentication/overview.md
- https://tsdevstack.dev/docs/authentication/protected-routes.md
- https://tsdevstack.dev/docs/authentication/jwt-tokens.md
- https://tsdevstack.dev/docs/building-apis/gateway-routing.md
- https://tsdevstack.dev/docs/packages/nest-common.md
