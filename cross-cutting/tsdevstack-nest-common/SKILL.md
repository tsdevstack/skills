---
name: tsdevstack-nest-common
description: Use when writing tsdevstack backend (NestJS) code and deciding how to do bootstrap, secrets, database, Redis, logging, metrics, rate-limiting, or email. The @tsdevstack/nest-common package provides these — use them, never re-implement or reinstall equivalents.
---

# nest-common — the tsdevstack backend building blocks

Everything cross-cutting lives in `@tsdevstack/nest-common`. Models know NestJS; what's non-obvious here is that you must **use these instead of installing your own.** All modules are global — import once.

## Bootstrap (don't hand-roll NestFactory)

- Services start with `startApp(AppModule)`; detached workers with `startWorker(WorkerModule)`. These wire Swagger, security, versioning, health, and the global `AuthGuard`. **Do not call `NestFactory.create()` yourself.**

## Use these, never reinstall

- **Secrets:** `SecretsService.get('KEY')`, never `process.env` — see the `tsdevstack-secrets` skill.
- **Database:** `createPrismaConnection()` (pooled pg adapter, auto `DB_POOL_MAX`) — never `new PrismaClient()`.
- **Redis:** `RedisModule` / `RedisService`.
- **Auth:** global `AuthGuard`, `@Public`, `@PartnerApi`, `@Partner` — see the `tsdevstack-auth-model` skill.
- **Observability:** `ObservabilityModule` (one import for logging, metrics, tracing, health); inject `LoggerService` (use `.child('Ctx')`) and `MetricsService`.
- **Rate limiting:** `RateLimitModule` + `@RateLimit(...)`.
- **Email / notifications:** `NotificationModule` / `NotificationService`.
- **Background jobs:** `BullConfigModule`, `SchedulerGuard`.
- **Service-to-service:** `BaseServiceClient` + `filterForwardHeaders`.
- **Pub/sub** → the `tsdevstack-messaging` skill. **Object storage** → the `tsdevstack-storage` skill.

## Never

- Install a library nest-common already provides (auth, Redis, logging, metrics, rate-limit, BullMQ, notifications, Prisma pooling).
- Read secrets from `process.env`, create your own `PrismaClient`, or bootstrap Nest by hand.

## Reference

- https://tsdevstack.dev/docs/packages/nest-common.md
