---
name: superb-ai-label-and-version-project
description: Set up a Superb AI labeling project over an existing dataset, assign and review work in bulk, then freeze a version for training or export.
api: Superb AI MLOps Platform API
base_url: https://api.bdai.superb-ai.com
operations:
  - projects-create_project
  - projects-create_class
  - projects-create_attribute
  - projects-add_member
  - projects-project_bulk_add_assets
  - projects-project_bulk_assign
  - projects-project_bulk_approve
  - projects-project_bulk_reject
  - projects-project_bulk_set_status
  - projects-get_project_progress
  - annotations-project_bulk_create_annotations
  - projects-create_version
  - jobs-get_job
generated: '2026-08-29'
method: generated
source: openapi/superb-ai-mlops-platform-openapi.json
---

# Set up, label and version a Superb AI project

A **project** labels exactly one dataset. A **version** is a frozen snapshot of that project's
labels — it is what training and export consume.

## Steps

1. **Create the project** — `projects-create_project`
   `POST /tenants/{slug}/projects` — takes the `dataset_id` of the dataset it will label.

2. **Define the label schema** — `projects-create_class`, then `projects-create_attribute`
   `POST /tenants/{slug}/projects/{project_id}/classes` and `.../classes/{class_id}/attributes`
   Do this before any labeling. Once a class is in use it locks: editing or deleting it returns
   **409 `CLASS_IN_USE`** or **`CLASS_LOCKED`**.

3. **Add people** — `projects-add_member`
   `POST /tenants/{slug}/projects/{project_id}/members`

4. **Pull assets into the project** — `projects-project_bulk_add_assets`
   `POST /tenants/{slug}/projects/{project_id}/assets/bulk-add`
   Async, returns a job. Send an `Idempotency-Key`.

5. **Assign work** — `projects-project_bulk_assign` (and `..._bulk_unassign` to undo)
   `POST /tenants/{slug}/projects/{project_id}/assets/bulk-assign`

6. **Import existing annotations, if any** —
   `annotations-project_bulk_create_annotations`
   `POST /tenants/{slug}/projects/{project_id}/annotations/bulk-create`
   Accepts an `Idempotency-Key`. For a large external import use
   `annotations-annotation_import_init` instead.

7. **Review** — `projects-project_bulk_approve`, `projects-project_bulk_reject`, or
   `projects-project_bulk_set_status` for an explicit workflow state.

8. **Track progress** — `projects-get_project_progress`
   `GET /tenants/{slug}/projects/{project_id}/progress`

9. **Freeze a version** — `projects-create_version`
   `POST /tenants/{slug}/projects/{project_id}/versions`
   This is the handoff point to `superb-ai-export-training-data` and
   `superb-ai-train-and-deploy-model`.

## Rules

- **Bulk vs batch.** Operations come in `bulk-*` (async, returns a job, accepts
  `Idempotency-Key`) and `batch-*` (synchronous) forms. Prefer `bulk-*` for anything large and
  poll `jobs-get_job`; **`JOB_PER_TENANT_CAP`** means too many jobs are already running, so wait
  rather than retrying immediately.
- **Locked shapes are never destroyed.** `projects-project_bulk_delete_annotations` skips locked
  annotations and reports them as `skipped_locked` in the job's `result_summary`. Read that
  field — a "successful" job may have changed less than you asked.
- **Project state gates writes.** **`PROJECT_ARCHIVED`** and **`PROJECT_STATE_FORBIDDEN`** mean
  the action is refused because of state, not permissions.
- **`projects-delete_project` and `projects-delete_version` are HARD deletes.** The contract says
  there is no archive and no undo, and deleting a project cascades away every version, model,
  deployment and export beneath it. Never call either without explicit human approval.
