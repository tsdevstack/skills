---
name: tsdevstack-nest-common
description: Use when writing tsdevstack backend (NestJS) code and deciding how to do bootstrap, secrets, database, Redis, logging, metrics, rate-limiting, roles, or email. The @tsdevstack/nest-common package provides these; use them, never re-implement or reinstall equivalents.
---

# nest-common: the tsdevstack backend building blocks

Everything cross-cutting lives in `@tsdevstack/nest-common`. Models know NestJS; what's non-obvious here is that you must **use these instead of installing your own.** All modules are global: import once.

## Bootstrap (don't hand-roll NestFactory)

- Services start with `startApp(AppModule)`; detached workers with `startWorker(WorkerModule)`. These wire Swagger, security, versioning, health, and the global `AuthGuard`. **Do not call `NestFactory.create()` yourself.**

## Use these, never reinstall

- **Secrets:** `SecretsService.get('KEY')`, never `process.env`. See the `tsdevstack-secrets` skill.
- **Database:** `createPrismaConnection()` (pooled pg adapter, auto `DB_POOL_MAX`), never `new PrismaClient()`.
- **Redis:** `RedisModule` / `RedisService`. It reconnects on its own after an outage; `isReady()` tells you whether commands can run now, `onReady(fn)` runs `fn` on every (re)connect. Commands fail fast while disconnected, so handle errors. The health check reports Redis `down` during an outage.
- **Auth:** global `AuthGuard`, `@Public`, `@PartnerApi`, `@Partner`, `@ApiKey` (the partner key's `{ id, consumer }`), types `AuthenticatedRequest`, `AuthType`, `AuthenticatedApiKey`. See the `tsdevstack-auth-model` skill.
- **API key index (Redis contract):** `hashApiKey`, `buildApiKeyRecordKey`, `encodeApiKeyRecord`, `decodeApiKeyRecord`, `validateApiKeyRecord`, `getApiKeyRecordExpireAt`, window helpers, `API_KEY_*` constants, type `ApiKeyRecord`. Only for writing API key records to Redis yourself (projects without the auth template); the auth service already does it. See https://tsdevstack.dev/docs/authentication/api-keys.md.
- **Roles:** `@Roles('ADMIN')` (applies `RolesGuard`; checks `systemRole` and custom `roles` from the JWT, 403 otherwise). `ROLES_KEY` is the metadata key if you build your own guard.
- **Observability:** `ObservabilityModule` (one import for logging, metrics, tracing, health); inject `LoggerService` (use `.child('Ctx')`) and `MetricsService`.
- **Rate limiting:** `RateLimitModule` + `@RateLimit(...)`. Per-IP limits use the client IP Kong reports (`X-Real-IP`) only on requests that came through the gateway; direct calls are limited by socket address. The `userId` limiter counts partner keys by key id.
- **Email / notifications:** `NotificationModule` / `NotificationService`.
- **Background jobs:** `BullConfigModule`, `SchedulerGuard`.
- **Service-to-service:** `BaseServiceClient` + `filterForwardHeaders`.
- **Pub/sub** → the `tsdevstack-messaging` skill. **Object storage** → the `tsdevstack-storage` skill.

## Never

- Install a library nest-common already provides (auth, Redis, logging, metrics, rate-limit, BullMQ, notifications, Prisma pooling).
- Read secrets from `process.env`, create your own `PrismaClient`, or bootstrap Nest by hand.

## Reference

- https://tsdevstack.dev/docs/packages/nest-common.md
