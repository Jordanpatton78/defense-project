# Tagup Defense Project

Ranks the severity of risk descriptions on a 0-10 scale. A CSV in Google Cloud Storage goes in, and each row comes back with a `severity_score` produced by a small open-source language model reading that row's `description` column.

This is a copy of the AI-engineering-base repository, trimmed to the stages this project needs. The rest of the base code is still in the repository but is not wired into the workflow.

## Status

Work in progress. The pipeline runs end to end, but the scores are not yet usable:

- **Done:** ingestion from GCS, the Cloud Run Job and Workflow wiring, and the inference config that points at a Hugging Face model.
- **Not yet built:**
  - A prompt template, so the model is asked for a 0-10 score instead of continuing the text.
  - Score parsing, so the model's reply becomes an integer clamped to 0-10 and records that don't parse are marked failed.
  - Left padding for batched generation. Until this is fixed, `batch_size` is 1, which is correct but slow.

Until the first two are done, `severity_score` holds raw model text. A successful run shows the pipeline works, not that the scores are meaningful.

## Pipeline

| Stage | Script | What it does |
|---|---|---|
| 1. Ingestion | `pipeline/sample.py` | Reads the CSV from `gs://` and writes `canonical.jsonl` plus a manifest. Every column is kept. |
| 6. Inference | `pipeline/inference.py` | Runs the language model on each record's `description` and writes `predictions.jsonl`. Failed records are kept and marked, not dropped. |

The stage numbers match the base project, so output folders are `stage_1` and `stage_6`.

The workflow (`pipeline/workflow.yaml`) runs the two stages in order as Cloud Run Jobs. Each run writes to `gs://<ARTIFACT_BUCKET>/runs/<RUN_ID>/`.

## Configuration

Settings live in `pipeline/config/`.

- `inference_config.json` names the model and the input field:
  - `model_path`: the Hugging Face model, currently `Qwen/Qwen2.5-1.5B-Instruct`.
  - `text_field`: the column the model reads, currently `description`.
  - `max_new_tokens`, `max_length`, `batch_size`: generation limits.
  - `prediction_field`: where the output is written, currently `severity_score`.

The configs are copied into the container image at build time. After changing one, rebuild and redeploy before running.

## Running it

Run these from the project root.

```bash
make test                      # offline test suite (some contract tests are expected to fail)
make build push                # build the container image and push it to Artifact Registry
make deploy-all deploy-workflow
make run-workflow SOURCE_URI=gs://YOUR-BUCKET/path/to/data.csv RUN_ID=severity-001
make executions                # list workflow runs
make logs                      # show job logs
```

`SOURCE_URI` must point at a CSV with a `description` column. Each run writes to its own `RUN_ID` folder, so reruns don't overwrite earlier output.

### Requirements

You need an account with access to the project. Building the image needs `roles/serviceusage.serviceUsageConsumer`, `roles/cloudbuild.builds.editor`, and access to the Cloud Build source bucket. Pushing needs `roles/artifactregistry.writer`. Deploying needs `roles/run.admin`, `roles/workflows.editor`, and `roles/iam.serviceAccountUser`.

## Repository layout

```
pipeline/                   stage scripts, shared helpers, workflow, Dockerfile
pipeline/config/            JSON config (inference, plus unused label and train configs)
tests/                      pytest suite from the base project (see tests/README.md)
smoke/                      small sample dataset
Makefile                    build, deploy, and run targets
PRD.md, QRD.md, ERD.md      requirements and data model from the base project
```

## Open questions

- Which column in the source file has the ground-truth severity, if any? Without one, the scores can't be checked against known values.
- The base project's test suite pins the seven-stage workflow. Those tests will fail until they're updated for this two-stage version.
