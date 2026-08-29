---
name: superb-ai-train-and-deploy-model
description: Train a model on a frozen Superb AI version, deploy it to an inference endpoint, and run predictions — including against the ZERO foundation models.
api: Superb AI MLOps Platform API
base_url: https://api.bdai.superb-ai.com
operations:
  - training-get_training_catalog
  - training-create_training_run
  - training-get_training_run
  - training-get_training_metrics
  - training-cancel_training_run
  - models-list_models
  - models-get_model
  - deployments-deploy_model
  - deployments-get_deployment
  - deployments-predict
  - deployments-stop_deployment
  - foundation-models-list_foundation_models
  - foundation-models-predict
  - auto-label-create_run
  - auto-label-cancel_run
generated: '2026-08-29'
method: generated
source: openapi/superb-ai-mlops-platform-openapi.json
---

# Train, deploy and call a model on Superb AI

The chain is: **frozen version → training run → model → deployment → predict.** Each arrow is a
separate resource; a training run is not a model and a model is not an endpoint.

## Steps

1. **See what you can train** — `training-get_training_catalog`
   `GET /tenants/{slug}/training/catalog`

2. **Start a training run** — `training-create_training_run`
   `POST /tenants/{slug}/projects/{project_id}/versions/{version_id}/training-runs`

3. **Watch it** — `training-get_training_run` and `training-get_training_metrics`
   To stop early, `training-cancel_training_run`. **Runs are not deletable** — cancelling sets a
   terminal status, it does not remove the record.

4. **Find the produced model** — `models-list_models` / `models-get_model`
   `GET /tenants/{slug}/models`

5. **Deploy it** — `deployments-deploy_model`
   `POST /tenants/{slug}/models/{model_id}/deployments`
   Poll `deployments-get_deployment`. **`MODEL_LOADING`** and **`MODEL_STARTING`** are expected
   transient codes while the endpoint warms — keep polling; do not redeploy.
   **`RESOURCE_NOT_READY`** means the same thing at the resource level.

6. **Predict** — `deployments-predict`
   `POST /tenants/{slug}/deployments/{deployment_id}/predict`

7. **Stop it when idle** — `deployments-stop_deployment`
   `DELETE /tenants/{slug}/deployments/{deployment_id}`
   A running deployment consumes paid compute. Stopping is the reversal path for step 5.

## Using ZERO without training anything

`foundation-models-list_foundation_models` (`GET /tenants/{slug}/foundation-models`) lists the
available foundation models, and `foundation-models-predict`
(`POST /tenants/{slug}/foundation-models/{key}/predict`) runs open-vocabulary detection directly.
The request schema (`ZeroPredictRequest`) takes an image plus `text_prompts` — open-vocabulary
class names to detect — and optional `box_prompts`. No training run, model or deployment is
needed for this path.

## Auto-labeling

`auto-label-create_run` (`POST /tenants/{slug}/projects/{project_id}/auto-label/runs`) applies a
model across a project. Configure it first with `auto-label-put_config`.

## Rules

- **Auto-label and training runs spend money.** The contract says the cancel path exists precisely
  because a run "spends inference $". `auto-label-cancel_run` sets a terminal `canceled` status
  that the worker honours **at the next chunk boundary** — cancellation is not instantaneous, so
  expect some further spend after the call returns.
- **`models-delete_model` is a hard delete** with no soft-delete or archive, and it cascades to
  the model's deployment rows. Escalate to a human before calling it.
- **Idempotency.** `auto-label-create_run` accepts an `Idempotency-Key` header — always set one,
  because a duplicate run is a duplicate bill. `training-create_training_run` does **not** accept
  the header, so guard it on your side: check `training-list_training_runs` before retrying a
  request whose response you did not see.
- **`JOB_ALREADY_RUNNING`** and **`JOB_TERMINAL`** distinguish "wait" from "this is over".
- Expect **429 `RATE_LIMITED`** (per-tenant token bucket) with no published `Retry-After`.
