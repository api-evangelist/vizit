---
name: Bulk export an organization's scores
description: Queue an asynchronous CSV export of PDP or image scores, poll until COMPLETED, and download the artifact before its 7-day retention expires.
api: https://docs.vizit.com/api
base_url: https://ext.vizit.com
operations:
  - createToken
  - createExport
  - getExport
  - downloadExport
generated: '2026-08-02'
method: generated
source: https://docs.vizit.com/guides/export-scores
---

# Bulk export an organization's scores

Use this to pull every PDP or every image Vizit has scored for your organization as a
single CSV, instead of polling per-product endpoints. The flow is always
**`POST` → poll `GET` → `GET` download**.

## Step 1 — Get a Bearer token (`createToken`)

`POST /auth/token` with the client-credentials body. Cache until `expires_in` elapses.
Send `X-Request-Id` on every call so your logs correlate with Vizit's.

## Step 2 — Create the export (`createExport`)

`POST /v1/exports`

- `subject` — **required**, `"pdps"` or `"images"`. Anything else returns
  `400 INVALID_SUBJECT`.
- `filters` — optional object; unknown keys return `400 INVALID_FILTER`.
  - `portfolio_id` (uuid) — restrict to one portfolio. Outside your org returns
    `404 PORTFOLIO_NOT_FOUND` (404, not 403, so cross-org existence never leaks).
  - `retailer_region` — e.g. `amazon_us`. Unsupported values return
    `400 INVALID_RETAILER_REGION`.
  - `last_refreshed_after` — RFC 3339 timestamp; this is the "give me what changed since
    my last sync" filter. Malformed values return `400 INVALID_TIMESTAMP`.

Org scope is enforced by the token; filters only narrow within your org.

Returns `202 Accepted` with `export_id`, `subject`, `status: "PROCESSING"`, `status_url`
and `requested_at`. Poll `status_url`.

**One in-flight export per organization.** A second submission while one is `PROCESSING`
returns `409 EXPORT_CONCURRENCY_LIMIT` — poll the existing job instead of retrying.

## Step 3 — Poll the job (`getExport`)

`GET /v1/exports/{export_id}`

Lifecycle:

| State | `download_url` | `error_code` |
|---|---|---|
| `PROCESSING` | `null` | none |
| `COMPLETED`, within retention | populated | none |
| `COMPLETED`, past 7-day retention | `null` | `EXPORT_EXPIRED` |
| `ERROR` | `null` | `EXPORT_FAILED` |

On `COMPLETED` you also get `completed_at`, `row_count`, `byte_size`,
`download_url_expires_at` and `expires_at` (= `completed_at` + 7 days). There is no
separate URL-level TTL — the download endpoint stays valid as long as the artifact is
retained.

A foreign `export_id` returns `404 EXPORT_NOT_FOUND`, indistinguishable from a missing
one.

## Step 4 — Download the artifact (`downloadExport`)

`GET /v1/exports/{export_id}/download` — this is exactly what `download_url` points at.

Authenticate with the **same Bearer token**; there is no presigned URL and no embedded
credential, so the path is safe to log or pass around inside your own systems.

Response: `Content-Type: text/csv`, `Content-Disposition: attachment;
filename="<subject>-<timestamp>.csv"`, body is raw UTF-8 CSV with a header row and `\n`
line separators.

Every failure here is a `404` with a typed `error_code`:

- `EXPORT_NOT_FOUND` — not in your org.
- `EXPORT_NOT_READY` — still `PROCESSING`; keep polling step 3.
- `EXPORT_EXPIRED` — purged past 7-day retention; re-queue the export.

## Column sets

`subject=pdps`: `pdp_id`, `portfolio_id`, `retailer_region`, `item_id`, `name`, `brand`,
`category_id`, `category`, `listing_score`, `score_penalty`, `image_count_penalty`,
`image_mix_penalty`, `image_order_penalty`, `high_scoring_asset_count`,
`is_ideal_content_mix`, `images_matching_ideal_mix`, `score_status`,
`shopper_clarity_status`, `image_quality_status`, `composition_status`,
`product_staging_status`, `created_at`, `updated_at`, `last_refreshed`.

`subject=images`: `image_id`, `pdp_id`, `portfolio_id`, `retailer_region`, `item_id`,
`image_type`, `image_url`, `position`, `source`, `vizit_score`, `raw_score`,
`hero_score`, `classification`, `created_at`, `updated_at`, `scored_at`.

Two traps worth coding around:

1. `hero_score` is populated **only** on hero rows; carousel rows emit an empty value.
2. The three `*_penalty` columns carry the underlying score **multiplier** here, while
   the realtime PDP endpoints expose them as true/false flags. Do not compare them
   directly.

The header row is fixed per `subject`, so you can rely on a stable column set. Exports
are CSV in v1 — JSONL and Parquet are deferred to v2.

## Retries

Retry `429` and `5xx` with exponential backoff (start ~5s, double, cap ~60s), honoring
`Retry-After`. Do not retry other `4xx` until the request is fixed.
