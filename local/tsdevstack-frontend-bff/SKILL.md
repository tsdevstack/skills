---
name: tsdevstack-frontend-bff
description: Use when working on the Next.js frontend in a fullstack-auth tsdevstack project — calling the backend, handling auth/session, or wiring the generated API client. The frontend is a BFF and never calls backend services directly.
---

# Frontend BFF model (fullstack-auth)

The Next.js app is a Backend-for-Frontend. The browser calls only the app's own `/api/*` route handlers; those call the backend through the generated client → Kong. **The browser never calls a backend service directly.**

## Calling the backend

- Use the generated client (`@shared/{service}-client`), not hand-written `fetch`.
- Base URL is context-dependent: **`KONG_INTERNAL_URL`** server-side (route handlers / SSR — bypasses the load balancer and WAF) vs **`API_URL`** for browser-facing calls. Set both locally.
- Every server-side Kong client gets two request interceptors, as in `lib/externalApi/auth-service.api.ts`: `forwardClientIp` (the visitor's address and the frontend token, so Kong rate-limits per visitor and keeps the frontend's trace) and `injectTraceContext` from `@tsdevstack/telemetry` (passes the trace on). A new client without them counts every visitor as one and starts a new trace at Kong.
- Log a failed call with `describeError(error)` (`lib/utils/describe-error.ts`): method, path, status, message. **Never log the error object**: an Axios error carries the request body (passwords) and headers (tokens, the frontend token).
- The app answers `GET /health` with 200 (`app/health/route.ts`): the AWS load balancer, Azure App Service and Front Door check it. Keep the route.

## Auth / session

- Tokens live in **HttpOnly cookies** — never `localStorage` or JS-readable state. Set and clear them with `setTokenCookies` / `clearTokenCookies` (`lib/utils/cookies.ts`): `httpOnly`, `secure` in prod, `sameSite=lax`.
- A route handler reads the cookie and forwards `Authorization: Bearer …` to Kong. The frontend treats the JWT as opaque — it never validates it (validation is Kong's job; see the `tsdevstack-auth-model` skill).

## Endpoint separation

Browser → your `/api/*` handler (input validation, cookie handling, bot check) → generated client → Kong → backend. Keep API keys and secrets server-side in the handler; never ship them to the browser.

## Reference

- https://tsdevstack.dev/docs/authentication/session-management.md
- https://tsdevstack.dev/docs/building-apis/gateway-routing.md
