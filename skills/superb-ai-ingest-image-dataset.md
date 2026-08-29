---
name: superb-ai-ingest-image-dataset
description: Create a Superb AI image dataset and upload image assets into it using the presigned two-phase upload, then confirm the assets landed.
api: Superb AI MLOps Platform API
base_url: https://api.bdai.superb-ai.com
operations:
  - image_datasets-create_image_dataset
  - image_assets-upload_init
  - image_assets-upload_init_batch
  - image_assets-upload_complete
  - image_assets-list_assets
  - image_datasets-get_image_dataset_stats
  - jobs-get_job
generated: '2026-08-29'
method: generated
source: openapi/superb-ai-mlops-platform-openapi.json
---

# Ingest an image dataset into Superb AI

Every path below is under `https://api.bdai.superb-ai.com` and every operationId is verified
against the published OpenAPI document. `{slug}` is the workspace slug, not a UUID.

## Before you start

- Authenticate with a Bearer token (`Authorization: Bearer <token>`). Obtain one through
  `auth-authorize` → `auth-callback`, or mint a per-user key with `api-keys-create_api_key`.
- Confirm which workspace you are acting in with `me-list_my_tenants`. Calling a `/tenants/{slug}`
  path you are not a member of returns **403 `AUTH_TENANT_MISMATCH`** — that is a membership
  problem, not a token problem, so do not retry with a new token.

## Steps

1. **Create the dataset** — `image_datasets-create_image_dataset`
   `POST /tenants/{slug}/datasets/images`
   A duplicate name returns **409 `DATASET_NAME_TAKEN`**. List first or pick a distinct name;
   retrying the same body will not resolve it.

2. **Initialise the upload** — `image_assets-upload_init` for one file, or
   `image_assets-upload_init_batch` for many
   `POST /tenants/{slug}/datasets/images/{dataset_id}/assets/upload-init[/batch]`
   This returns presigned upload targets. Prefer the batch form; one call per image will hit the
   per-tenant rate limiter quickly.

3. **Upload the bytes** to the presigned target returned in step 2. This is a direct object-store
   PUT, not a Superb AI API call — do not send your Bearer token to it.

4. **Complete the upload** — `image_assets-upload_complete`
   `POST /tenants/{slug}/datasets/images/{dataset_id}/assets/upload-init/{upload_id}/complete`
   An asset is not visible to the platform until this call succeeds. Watch for
   **`ASSET_FORMAT_UNSUPPORTED`**, **`ASSET_TOO_LARGE`** and **`DATASET_ASSET_CAP_EXCEEDED`**.

5. **Verify** — `image_assets-list_assets` and `image_datasets-get_image_dataset_stats`
   Page with `?cursor=` until `next_cursor` is `null`. There are no total counts, so do not try
   to compute a page count.

6. **Optional — embed the dataset for similarity search** —
   `image_datasets-embed_image_dataset`
   `POST /tenants/{slug}/datasets/images/{dataset_id}/embed`
   Async: returns a job. Poll `jobs-get_job`. Accepts an `Idempotency-Key` header — set one.
   `EMBEDDER_UNAVAILABLE` and `EMBEDDER_INVOCATION_FAILED` are transient; `ASSET_NOT_EMBEDDED`
   later means this step never ran.

## Rules

- **Idempotency.** Send an `Idempotency-Key` header on the embed call and on every bulk POST. A
  replay is reported as **`IDEMPOTENCY_REPLAY`**, so a retry is safe and detectable.
- **Errors.** Branch on `error.code`, never on `error.message` — the spec says the enum is stable
  and the message is not. Log `error.request_id` for support.
- **Rate limits.** All operations can return **429 `RATE_LIMITED`** (per-tenant token bucket).
  No `Retry-After` or `RateLimit-*` header is published, so back off exponentially with jitter
  rather than reading a reset time.
- **Deletion is not reversible.** `image_datasets-delete_image_dataset` and
  `image_assets-bulk_delete` have no undo and no restore endpoint. Escalate to a human first.
