---
name: Score an image and request Spark improvements
description: Submit a standalone image for Visual AI scoring, read its GS1 hero component sub-scores and agent-ready flag, then request Spark Ideas and Spark Images variations for it.
api: https://docs.vizit.com/api
base_url: https://ext.vizit.com
operations:
  - createToken
  - scoreImage
  - getImageScore
  - createSparkIdea
  - getSparkIdea
  - createSparkImagesBatch
  - getSparkImagesBatch
generated: '2026-08-02'
method: generated
source: https://docs.vizit.com/api
---

# Score an image and request Spark improvements

Use this to evaluate a single image outside a full listing, then ask Vizit's generative
Spark surface for written suggestions and image variations.

## Step 1 — Get a Bearer token (`createToken`)

`POST /auth/token` with the client-credentials body. Cache until `expires_in` elapses.

## Step 2 — Submit the image (`scoreImage`)

`POST /v1/images/score`

Body (all required):

- `image_url` — publicly reachable HTTP/HTTPS, jpeg/png/webp, 25 MB or less.
- `product_category_id` — a category UUID from `listProductCategories`. Outside your ICP
  it is rejected with `CATEGORY_NOT_IN_ORG_ICP`.
- `image_type` — `hero` or `carousel`.

Returns `202 Accepted` with `image_id` and `status: PROCESSING`. Keep the `image_id`.

Carousel images that classify as an informational asset type are **not scored** — they
resolve to `error_code: IMAGE_OMITTED_FROM_SCORING`. Hero submissions are always scored.

## Step 3 — Poll for the score (`getImageScore`)

`GET /v1/images/{image_id}/score`. Poll until `status` is `COMPLETED` or `ERROR`.

On `COMPLETED` read:

- `vizit_score` — composite 0–100; `vizit_certified` is true at 80 or above.
- `classification` — the normalized label (e.g. `Lifestyle`, `Packshot`).
- For **hero** images only:
  - `hero_gs1_components` — per-component GS1 sub-scores: `four_ws.brand`,
    `four_ws.product_type`, `four_ws.variant`, `four_ws.size_and_count`,
    `image_quality.background_separation`, `image_quality.product_centered`,
    `image_quality.product_prominent`, `image_quality.product_cropping`,
    `image_quality.product_tilt`, `image_quality.no_off_pack_text`, `composition`,
    `layout_staging`.
  - `hero_gs1_component_statuses` — `PASS` / `REVIEW` / `FAIL` per component
    (`HIGH` / `MEDIUM` / `LOW` for `composition` and `layout_staging`).
  - `hero_gs1_group_statuses` — roll-ups for `shopper_clarity`, `image_quality`,
    `composition`, `product_staging`.
  - `agent_ready` — true when **every** `four_ws.*` shopper-clarity component scores 90
    or above; a component with no score counts as not ready. Always `null` for carousel
    images.
  - `mobile_ready` — same shopper-clarity rule as `agent_ready`. Not related to the
    retired `mobile_readiness` object.

`agent_ready` is the flag to gate on when you are preparing imagery for AI-agent shopping
surfaces.

## Step 4 — Request a Spark Idea (`createSparkIdea`)

`POST /v1/images/{image_id}/spark/ideas`

Only **root** images may seed Spark — a Spark-generated image returns
`400 GENERATED_IMAGE_NOT_ALLOWED`. The image must have a `product_category_id` assigned
or you get `400 MISSING_PRODUCT_CATEGORY`.

**Idempotent on `image_id`.** While an Idea is in flight for that image, another call
returns the existing Idea with `200 OK`; a fresh create returns `202 Accepted`. The
organization check runs *before* the idempotent lookup, so a caller from another
organization gets `404 IMAGE_NOT_FOUND` rather than reusing your in-flight Idea. Do not
add your own dedupe key — branch on the status code.

Keep `idea_id` and follow `result_url`.

## Step 5 — Poll the Idea (`getSparkIdea`)

`GET /v1/images/{image_id}/spark/ideas/{idea_id}`. Poll until `status` moves from
`PROCESSING` to `COMPLETED` or `ERROR`. `data` carries the suggestion **markdown** only
when `status` is `COMPLETED`; it is `null` in every other state.

An Idea that stays non-terminal past the 5-minute processing timeout is reported as
`ERROR` here even if the underlying record still reads as in-flight — so a stuck request
is always observable. Do not poll forever; treat 5 minutes as the ceiling.

## Step 6 — Request Spark Images (`createSparkImagesBatch`)

`POST /v1/images/{image_id}/spark/images` — same root-image rule, same idempotency on
`image_id` (`200` for reuse of an in-flight batch, `202` for a fresh one). Keep
`batch_id`.

## Step 7 — Poll the batch (`getSparkImagesBatch`)

`GET /v1/images/{image_id}/spark/images/{batch_id}`. `data` stays empty until `status` is
`COMPLETED`. The batch only completes once every prompt is terminal **and** Spark Cull has
resolved every child image; while cull settles it stays `PROCESSING` with empty `data`.

On `COMPLETED`, `data` holds the **visible children only**, each with a short-lived
pre-signed download URL — re-request the batch to refresh an expired URL rather than
caching the URL itself. Same 5-minute non-terminal timeout as Ideas.

## Errors to handle

| Code | Status | What to do |
|---|---|---|
| `NO_DEFAULT_PORTFOLIO` | 400 | Org has no default portfolio. |
| `CATEGORY_NOT_IN_ORG_ICP` | 400 | Category is outside your ICP. |
| `EMPTY_IMAGE` | 400 | The downloaded image body was empty. |
| `IMAGE_DOWNLOAD_FAILED` | 400 | The image URL could not be downloaded. |
| `GENERATED_IMAGE_NOT_ALLOWED` | 400 | Seed Spark from a root image, not a generated one. |
| `MISSING_PRODUCT_CATEGORY` | 400 | Assign a category to the image first. |
| `IMAGE_NOT_FOUND` | 404 | Not in your org (also returned for cross-org access). |
| `IDEA_NOT_FOUND` | 404 | The Idea is not linked to this image. |
| `IMAGE_OMITTED_FROM_SCORING` | — | The carousel image is an informational asset type; not an HTTP failure. |
| `SQS_ENQUEUE_FAILED` | 500 | The record was marked ERROR; resubmit. |
| `VALIDATION_ERROR` | 422 | Read `details[]`. |
