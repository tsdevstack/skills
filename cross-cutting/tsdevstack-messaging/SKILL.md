---
name: tsdevstack-messaging
description: Use when adding inter-service pub/sub to tsdevstack — one service emits an event and others react. Not for intra-service job queues (that is BullMQ).
---

# Messaging (Redis Streams pub/sub)

Inter-service pub/sub over the existing Redis — no new infrastructure. Every consumer group receives every message (broadcast). For one-consumer-per-job work, use BullMQ instead.

## Set up

- Declare topics via the CLI: `npx tsdevstack add-messaging-topic <name>` (also `update-messaging-topic` / `remove-messaging-topic`). Topics live in `.tsdevstack/config.json`.
- In the service: `MessagingModule.forRoot({ consumerGroup, topics })` from `@tsdevstack/nest-common`.

## Publish / consume

- **Publish:** inject the messaging service and call `publish(topic, payload)`.
- **Consume:** decorate a handler with `@OnMessage(topic)`. Return normally → auto-ack. Throw → retry (after ~60s idle), 3 failures → dead-letter queue (`{project}:messaging:{topic}:dlq`). Inspect the DLQ in Redis Commander.

## Reference

- https://tsdevstack.dev/docs/features/async-messaging.md
