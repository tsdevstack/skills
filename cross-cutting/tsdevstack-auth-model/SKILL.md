---
name: tsdevstack-auth-model
description: Use when adding or changing authentication on a tsdevstack endpoint, choosing between @Public / @ApiBearerAuth / @PartnerApi / @Roles, reading the authenticated user (req.user) or partner (req.apiKey), issuing or limiting partner API keys, wiring service-to-service calls, or debugging an unexpected 401/403/404 through the gateway.
---

# tsdevstack auth model

Auth is **two independent layers**. Getting the combination wrong is the single most common tsdevstack mistake. Kong validates the JWT at the edge; your backend never validates a JWT itself.

## The two layers

1. **Kong (gateway)**: controlled by `@ApiBearerAuth()`. Present ⇒ Kong's OIDC plugin requires and validates the JWT (RS256, via the auth service's JWKS) before the request reaches your service. Absent ⇒ Kong forwards the route with no JWT check.
2. **AuthGuard (backend, global)**: controlled by `@Public()`. It runs on every route (registered as `APP_GUARD` from `@tsdevstack/nest-common`; the class is `AuthGuard`). Absent ⇒ it requires a logged-in user or a valid service API key. Present ⇒ anyone may call it.

## The decorator matrix (get this right)

| Intent | `@ApiBearerAuth()` | `@Public()` |
|---|---|---|
| Logged-in users only | yes | no |
| Truly public | no | yes |
| **Bug: 401 in prod** | no | no |
| (rarely useful) | yes | yes |

**Omitting both is the classic bug:** Kong lets the request through (no JWT required), then the global `AuthGuard` rejects it (`No authentication provided`) → 401. A public route needs **both** `@Public()` *and* no `@ApiBearerAuth()`.

Decorators come from `@tsdevstack/nest-common` (`@Public`, `@PartnerApi`, `@Partner`, `@ApiKey`, `@Roles`); `@ApiBearerAuth` is the standard Swagger decorator. `generate-kong` builds Kong routes from the OpenAPI spec: one **exact** route per path with only its declared methods (plus `OPTIONS`). Anything not in the spec (other paths, extra segments, trailing slashes, other methods, `@ApiExcludeEndpoint()` routes) gets **404 from Kong**. After adding or changing an endpoint, regenerate the OpenAPI docs and the Kong config (`npx tsdevstack sync`).

## Reading the user

Type the request as `AuthenticatedRequest` (from nest-common). On JWT routes `req.authType === 'user'` and:

- `req.user.id`: the user id (the JWT `sub` claim). **Use `.id`, never `.sub`.**
- `req.user.<claim>`: any other claim (email, systemRole, roles, tenantId, …), typed dynamically, parsed to its real type.

Kong forwards the claims in `X-Userinfo`; your backend does **not** decode or validate the JWT, it just reads `req.user`. The old `X-JWT-Claim-*` headers are no longer read.

## Roles

- Auth template users have a `systemRole` (`USER` or `ADMIN`) and a list of custom `roles` (declared in the auth service's `src/roles/roles.constants.ts`), both in the JWT.
- `@Roles('ADMIN')` or `@Roles('ADMIN', 'BILLING')` on a handler or controller: the user needs at least one of them (system role or custom role), otherwise 403. Partner keys and service calls have no user and always get 403.
- Roles come from the token, so a change applies at the user's next token refresh. Re-check the database for sensitive actions.
- First admin: list emails in the `ADMIN_EMAILS` secret (auth service); a confirmed user in that list becomes `ADMIN` at login. Admins change roles through the auth service's admin API (`/auth/v1/admin/users`).

## Partner / API-key access

- `@PartnerApi()` gives that one operation an exact partner route at `/api` + its path (`/api/{prefix}/...`), with Kong checking the `x-api-key`. Combine with `@ApiBearerAuth()` for dual access (JWT at `/{prefix}/...`, key at `/api/{prefix}/...`).
- A partner key only reaches `@PartnerApi()` operations: other endpoints have no `/api` route (404 at Kong), and `AuthGuard` answers 403 if a key request lands on a handler without `@PartnerApi()`, even a `@Public()` one.
- On key requests: `req.authType === 'apiKey'`, `req.service === 'partner'`, `req.apiKey = { id, consumer }`, and `req.user` is never set. `@Partner()` returns `req.apiKey.consumer`; `@ApiKey()` returns `req.apiKey`.
- Kong does not forward the raw `x-api-key` to your backend; it sends `X-Api-Key-Id` and `X-Api-Key-Consumer`, which `AuthGuard` reads only with a valid trust token.

## API keys (key model)

- Keys are runtime data, not config or secrets. They belong to a named **consumer** (`acme-corp`), not a user. Admins manage them through the auth service's admin API (`/auth/v1/admin/api-keys`: create, list, get, update, revoke, rotate, usage). The raw key (`tsk_...`) is shown once; only its hash is stored.
- Kong's `tsdevstack-api-key` plugin checks every request against a Redis index the auth service maintains: 401 (`api_key_missing`, `invalid_api_key`, `api_key_revoked`, `api_key_expired`), 429 (`rate_limit_exceeded`, `quota_exceeded`), 503 (`index_unavailable`, `gateway_unavailable`; it fails closed). Changes apply on the next request, no redeploy.
- Per-key limits: minute, hour, day, week, month (UTC). A window without its own limit uses the global `rate-limiting` value from `kong.user.yml`. A per-IP ceiling (`framework.apiKeys.ipLimitPerMinute`, default 600) runs first on partner routes.
- Never add `consumers` / `keyauth_credentials` to `kong.user.yml` or put partner keys in `.secrets.user.json`: static keys were removed and no route checks them.
- Auth-template projects need the `sync-api-key-usage` scheduled job in every cloud environment (`infra:generate` prints the entry).
- Without the auth template, the project writes key records to Redis itself (documented contract; helpers exported by nest-common).

## Service-to-service

Each backend receives `{SERVICE}_SERVICE_API_KEY` + `{SERVICE}_URL` for its peers, plus its own `API_KEY`. Use `BaseServiceClient` + `filterForwardHeaders()` from nest-common. Direct calls send `x-api-key` + `x-service-name` and bypass Kong; `AuthGuard` validates the key against the callee's `API_KEY` (`req.authType === 'service'`). A partner request whose headers you forward downstream still counts as a partner there: the downstream handler needs `@PartnerApi()` too.

## Trust boundary (trust first)

Kong stamps `X-Kong-Trust` (= the `KONG_TRUST_TOKEN` secret) on every proxied request. `AuthGuard` checks it **before** reading any identity header:

- Valid token: the caller is a user (`X-Userinfo`) or a partner key, as Kong reported it.
- No token: identity headers are ignored. Only a valid service `x-api-key` or a `@Public()` handler works; anything else is 401. Calling `localhost:300x` directly with a hand-made `X-Userinfo` gets 401.
- Wrong token: 401.
- `/.well-known/*`, `/health`, `/metrics` (infra probes) ignore the token.

Kong also removes identity headers sent by clients (`X-Userinfo`, `X-Consumer-*`, `X-Api-Key-*`, `X-Kong-Trust`, …) before any auth plugin runs, so only Kong-set values reach your service.

## Templates

`framework.template` in `.tsdevstack/config.json`:

- `fullstack-auth`: auth service + Next.js BFF.
- `auth`: auth service only.
- `null`: no auth service; bring external OIDC via the `OIDC_DISCOVERY_URL` secret.

JWT signing keys, roles and `ADMIN_EMAILS` exist only for `auth` / `fullstack-auth`.

## Never

- Validate or decode JWTs in your backend: Kong does it.
- Read auth values from `process.env`: `AuthGuard` and `SecretsService` handle them.
- Assume a route is protected without `@ApiBearerAuth()`: Kong is what enforces the JWT.
- Read `req.user` on partner requests: use `@Partner()` / `req.apiKey`.

## Reference

- https://tsdevstack.dev/docs/authentication/overview.md
- https://tsdevstack.dev/docs/authentication/protected-routes.md
- https://tsdevstack.dev/docs/authentication/jwt-tokens.md
- https://tsdevstack.dev/docs/authentication/api-keys.md
- https://tsdevstack.dev/docs/building-apis/gateway-routing.md
- https://tsdevstack.dev/docs/packages/nest-common.md
