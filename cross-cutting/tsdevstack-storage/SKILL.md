---
name: tsdevstack-storage
description: Use when adding object storage (file upload/download, presigned URLs) to a tsdevstack service. The same code runs on local MinIO and cloud S3/GCS/Blob — never install a cloud SDK directly.
---

# Object storage

One API across providers. Local = MinIO; the cloud adapter is selected from `SECRETS_PROVIDER` (there is **no** separate `STORAGE_PROVIDER`). **Never install `@aws-sdk`, `@google-cloud/*`, or `@azure/*` directly** — use the module.

## Set up

- Add a bucket via the CLI: `npx tsdevstack add-bucket-storage --name <name>` (also `remove-bucket-storage`), then `npx tsdevstack sync`.
- In the service: `StorageModule.forRoot({ buckets })` from `@tsdevstack/nest-common`; inject with `@InjectStorage(bucket)`.

## Use

`upload`, `download`, and presigned URLs (`getPresigned…`) via the injected provider. Presigned uploads go straight to storage, bypassing Kong. Buckets are always private.

## Reference

- https://tsdevstack.dev/docs/features/object-storage.md
