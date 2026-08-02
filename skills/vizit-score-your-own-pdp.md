---
name: Score a product detail page from your own images
description: Resolve a Vizit product category, upsert a PDP under your own stable identifier with hero and carousel image URLs from your PIM, and poll for listing scores.
api: https://docs.vizit.com/api
base_url: https://ext.vizit.com
operations:
  - createToken
  - listProductRetailers
  - listProductCategories
  - upsertPdpById
  - getPdpById
generated: '2026-08-02'
method: generated
source: https://docs.vizit.com/api
---

# Score a product detail page from your own images

Use this when you control the imagery — for example syncing from a PIM or DAM — and want
Vizit to score a listing under **your** identifier rather than an Amazon ASIN.

## Step 1 — Get a Bearer token (`createToken`)

`POST /auth/token` with `client_id`, `client_secret`, `audience`
(`https://ext.vizit.com`), `organization` and `grant_type: "client_credentials"`. Cache
the token until `expires_in` elapses.

## Step 2 — Resolve the retailer region (`listProductRetailers`)

`GET /v1/product-retailers` returns the retailer regions you may use. Each entry has
`id` (e.g. `amazon_us`, `walmart_us`), `retailer`, `region`, `display_name` and `domain`.
Use the `id`.

## Step 3 — Resolve a category (`listProductCategories`)

`GET /v1/product-categories?retailer=amazon_us` lists the categories your organization
has ICP access to: `id`, `category_name`, and a hierarchical `path` separated by `" > "`.
An invalid `retailer` value returns `400`.

Take the category `id` and use it as `product_category_id` in the next step. A category
outside your ICP is rejected with `CATEGORY_NOT_FOUND`.

Category is optional if you would rather have Vizit infer it — see step 4.

## Step 4 — Upsert the PDP (`upsertPdpById`)

`PUT /v1/pdps/id/{id}` where `{id}` is **your own stable identifier** (max 512
characters; empty or longer returns `400 INVALID_ID`). It is echoed back on the GET so
you can correlate records.

Body fields:

- `hero_image_url` — **required**. Publicly reachable HTTP/HTTPS URL, jpeg/png/webp,
  25 MB or less.
- `carousel_image_urls[]` — ordered; order becomes carousel position. Maximum 20, or the
  request returns `413`.
- Exactly one of `product_category_id` (Vizit category UUID) or `external_category_id`
  (your own category id, mapped on your behalf) — supplying both is rejected. Omit both
  and the category is resolved from `gtin`, `asin` or `name`.
- `gtin` / `asin` — used **only** for category resolution, never echoed back. Handled
  leniently: a malformed value is ignored rather than rejected.
- `retailer` — a `retailer_regions` value; defaults to `amazon_us`.
- `name` — optional display name.
- `integration_id`, `hero_image_asset_id`, `carousel_image_asset_ids[]` — partner-side
  correlation identifiers, echoed back. `carousel_image_asset_ids` must be empty or the
  same length as `carousel_image_urls`.

**Idempotency.** This is an upsert with no `Idempotency-Key` header — the identifier in
the path is the key.

- `202 Accepted` + `status: PROCESSING` — the submission changed the PDP and it will be
  processed.
- `200 OK` + `status: COMPLETED` or `ERROR` — nothing changed, so nothing was
  reprocessed.

Carousel images are reconciled **by filename**: a filename already on the PDP is reused
with no re-download and no re-score; a new filename replaces that slot. An identical
payload is a no-op — safe to replay.

Use the returned `score_url` for polling; it already carries the `retailer` parameter.

## Step 5 — Poll for scores (`getPdpById`)

`GET /v1/pdps/id/{id}?retailer=amazon_us`, or follow `score_url`. Poll until `status` is
`COMPLETED` or `ERROR`. During a rescore the previous scores remain visible with
`status: PROCESSING` and the prior `scored_at`.

The score payload is the same shape as the ASIN flow: `pdp_score`, `vizit_certified`
(80+), `carousel_score`, `hero`, `carousel_images[]`, `asset_mix`,
`is_ideal_content_mix`, `images_matching_ideal_mix`, the four penalty flags,
`high_scoring_asset_count`, `listing_score_at_ingest` and `score_change`.

## Errors to handle

| Code | Status | What to do |
|---|---|---|
| `INVALID_ID` | 400 | id is empty or over 512 characters. |
| `INVALID_RETAILER` | 400 | Use an id from `listProductRetailers`. |
| `NO_DEFAULT_PORTFOLIO` | 400 | Your org has no default portfolio — contact Vizit. |
| `CATEGORY_NOT_FOUND` | 400 | Category does not exist or is outside your ICP. |
| `EXTERNAL_CATEGORY_UNMAPPED` | 400 | Your category id could not be mapped. |
| `CATEGORY_UNRESOLVED` | 400 | No category supplied and none inferable — send one. |
| `IMAGE_DOWNLOAD_FAILED` | 400 | Read `extra.failed_urls` for the per-URL reason. |
| `VALIDATION_ERROR` | 422 | Read `details[]` — each entry is `{field, issue}`. |
| — | 413 | More than 20 carousel images. |
| — | 429 | Honor `Retry-After`. |
| — | 502 | Image upload to S3 failed; retry with backoff. |

Always send `X-Request-Id` and log the `request_id` returned in the error body — Vizit
support asks for it plus the UTC request time.
