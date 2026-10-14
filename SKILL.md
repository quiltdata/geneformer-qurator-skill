---
name: geneformer-embeddings
description: Generate NVIDIA BioNeMo Geneformer cell embeddings from an .h5ad dataset or Quilt package file, and publish the result as a Quilt package. Use when a user asks to embed, cluster, or run Geneformer on single-cell data.
allowed-tools: geneformer_embed_start, geneformer_embed_status
---

# Geneformer cell embeddings

Two tools, both real paid-GPU operations against a live SageMaker async
endpoint — never claim a result without actually calling them.

## 1. Start a job: `geneformer_embed_start`

Params:
- `input` (str) — `s3://` URI of an `.h5ad` file, or
  `quilt+s3://bucket#package=name&path=file.h5ad`
- `output_bucket` (str) — bucket for job files and the result package
- `output_package` (str) — package name the status check will publish to
- `max_cells` (int, optional) — cap on cells embedded; default embeds all

Before calling, confirm with the user:
- The exact cohort (donors, cell counts, filter criteria) if it wasn't already
  described earlier in the conversation.
- That running this is a paid GPU job they want to run now, if that hasn't
  already been approved.

The input must be a real `.h5ad` under 1 GB; larger files need the cohort
subset first, not a `max_cells` cap. On success this returns `{job_id,
output_location, inference_id, eta}`. Report the `job_id` and `eta` back to
the user — don't invent a completion time.

## 2. Check status / publish: `geneformer_embed_status`

Params:
- `job_id` (str) — from `geneformer_embed_start`
- `output_bucket` (str) — same bucket passed to start
- `output_package` (str) — package to create or update

Call this to check on a job you started. It returns one of:
- `pending` — SageMaker hasn't produced output yet. Wait and check again
  rather than assume failure.
- `failed` — with the SageMaker failure text. Report it verbatim; don't
  speculate about the cause unless asked.
- `done` — the embeddings and a `summary.json` were published to
  `output_package` in `output_bucket` (creating or updating it), with a
  catalog link to the new package revision. Give the user that link.

A `done` result from a real run is the only basis for saying embeddings were
generated — never present a precomputed or example result as the output of
calling this tool.

## Errors

- `not_h5ad`: the input isn't an `.h5ad` file — check the path/extension.
- `too_large`: input exceeds 1 GB — subset the cohort before starting, don't
  retry with `max_cells` on an oversized file.
- `bad_job_id` / `no_job`: the job_id is invalid or wasn't found in
  `output_bucket` — confirm it's the exact id `geneformer_embed_start`
  returned, and that `output_bucket` matches what was passed to start.
- `rejected`: the SageMaker endpoint itself rejected the job — report the
  error text from the tool, don't retry blindly.
