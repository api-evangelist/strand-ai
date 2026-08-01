---
name: Predict protein markers from an H&E slide
description: Upload a whole-slide image and run a Lattice inference job to get spatial protein-marker predictions, streaming progress and downloading OME-Zarr results.
api: openapi/strand-ai-openapi-original.json
operations: [createUpload, completeUpload, estimatePrediction, submitPrediction, streamJob, getJob, getJobResults, getJobResultFile]
---

# Predict protein markers from an H&E slide

Use the Strand Platform API to turn one H&E whole-slide image (WSI) into per-pixel predictions
for a panel of protein markers with the Lattice model.

## Auth
All requests use `Authorization: Bearer sk-strand-...` (organization-scoped). Base URL:
`https://app.strandai.com/api/v1`. Access is invite-only.

## Steps

1. **Initiate the upload** — `createUpload` with `{filename, fileSize, contentType}`. Optionally
   pass `contentSha256` (64 lowercase hex) so the platform can de-duplicate: if `existing: true`
   comes back, reuse the returned `uploadId` and skip the byte upload.
2. **Upload the bytes** — PUT the slide bytes to the returned `uploadUrl` (resumable). Skip when
   `existing: true`.
3. **Finalize** — `completeUpload` with the `id` (uploadId). This marks the upload `ready` and
   records `widthPx`/`heightPx`, which credit estimates need.
4. **Estimate cost** — `estimatePrediction` with `{uploadId, markers[]}`. Credits =
   `ceil(W/224) * ceil(H/224)` patches × number of markers. Check `estimatedCredits` against
   `orgBalance` before spending.
5. **Submit** — `submitPrediction` with `{uploadId, markers[], model?}`. `model` is optional
   (`v0.4` or `v0.5`; default `v0.5`). Returns `202` with a `jobId` and `reservedCredits`. Handle
   `402 insufficient_credits` (top up, retry) and `429` (honor `Retry-After` seconds — per-org
   concurrency cap).
6. **Track progress** — prefer `streamJob` (SSE `text/event-stream`, 15s heartbeats, closes on
   terminal status). Or poll `getJob` for `{status, progress, resultsAvailable}`.
7. **Download results** — once complete, call `getJobResults` for a signed OME-Zarr URL, or walk
   the zarr tree file-by-file with `getJobResultFile` (API-key authenticated, no GCS creds needed).
   Results load as AnnData (Python `strand-sdk`) or SpatialExperiment (R `strandai`).

## Rules
- Errors use the Strand envelope `{error, message, required?}` — not RFC 9457. See
  `errors/strand-ai-problem-types.yml`.
- Predictions are for research/hypothesis generation only — never for clinical diagnosis,
  treatment selection, or patient care.
- Failed jobs auto-refund reserved credits.
