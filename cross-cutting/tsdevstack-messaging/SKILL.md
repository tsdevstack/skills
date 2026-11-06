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
- A handler runs in the trace of the request that published the message.

## How long messages stay

- A message waits in Redis until the subscriber handles it. A subscribing service with `minInstances: 0` handles nothing while it's scaled down: keep subscribers at 1 or more (the default).
- A long handler keeps its message (a "still working" signal every 20 s), so another instance doesn't run it again. A crash hands it out again after ~60 s: make handlers safe to run twice.
- A stream keeps each message until every subscriber group has handled it. `maxLen` caps a topic only while it has no subscriber yet, and the DLQ. A subscriber that stays down grows its stream, and Redis never evicts: when it's full, every write fails.
- **Removing a subscriber:** remove the subscription, deploy the service, wait ~15 minutes (it deletes its own consumer group), then remove the service. A removed service's group would stop the trimming.

## Reference

- https://tsdevstack.dev/docs/features/async-messaging.md
