---
name: superb-ai-export-training-data
description: Export a frozen Superb AI project version as a COCO, YOLO or Delta bundle and download it before it is garbage-collected.
api: Superb AI MLOps Platform API
base_url: https://api.bdai.superb-ai.com
operations:
  - projects-list_versions
  - exports-create_export
  - exports-list_exports
  - exports-get_export_download
  - exports-delete_export
  - jobs-get_job
generated: '2026-08-29'
method: generated
source: openapi/superb-ai-mlops-platform-openapi.json
---

# Export a labeled version from Superb AI

Exports are produced from a **frozen version**, not from a live project. Create the version first
(`projects-create_version`, see `superb-ai-label-and-version-project`).

## Steps

1. **Pick the version** — `projects-list_versions`
   `GET /tenants/{slug}/projects/{project_id}/versions`

2. **Request the export** — `exports-create_export`
   `POST /tenants/{slug}/projects/{project_id}/versions/{version_id}/exports`
   `format` must be one of the three values in `ExportFormat`:
   - `coco` — MS-COCO object-detection annotation format
   - `yolo` — YOLO (Darknet) label format
   - `delta` — Superb AI's own bundle format
   `include_assets` (default `true`) controls whether source image files ship in the bundle.

   Returns **202** with a queued row. Two idempotency layers apply:
   - **Implicit** — `options_hash = sha256(format || canonical_options)` enforces at most one
     live export per `(version, format, options)`. On collision you get the existing row with
     **`X-From-Cache: true`** and **HTTP 200 instead of 202**. Treat 200 as success, not as an
     error, and do not create a second export to "force" a fresh one.
   - **Explicit** — supply an `Idempotency-Key` header.

3. **Poll to completion** — `jobs-get_job`, or re-read `exports-list_exports`
   `ExportStatus` moves through `pending` → `queued` → `running` → `ready`, or lands on `failed`
   or `expired`. `size_bytes` and `exported_count` stay `null` until the bundle is complete.
   Check `skipped_count` on the finished row — a `ready` export can still have skipped assets.

4. **Download** — `exports-get_export_download`
   `GET /tenants/{slug}/projects/{project_id}/versions/{version_id}/exports/{export_id}/download`

## Rules

- **There is a real expiry window: 7 days.** The contract states a lifecycle rule garbage-collects
  export objects "within 7 days regardless", and both **`EXPORT_EXPIRED`** (error code) and
  `expired` (an `ExportStatus` value) exist. Download promptly; after expiry the only remedy is to
  create the export again.
- **`EXPORT_FAILED`** is terminal for that row — create a new export rather than polling on.
- **`exports-delete_export`** is workspace-admin only and force-deletes the row. The S3 object is
  removed by the lifecycle rule, not immediately.
- **Standards note.** Because `coco` and `yolo` are declared in the contract's own enums, a
  pipeline that already reads either format consumes a Superb AI export with no bespoke
  converter. See `conformance/superb-ai-conformance.yml`.
- Branch on `error.code`; expect **429 `RATE_LIMITED`** with no `Retry-After` header.
