---
name: Score an Amazon listing by ASIN
description: Authenticate against the Vizit Public API, submit an Amazon ASIN for Vizit's scrape-and-score pipeline, and poll until the listing score is available.
api: https://docs.vizit.com/api
base_url: https://ext.vizit.com
operations:
  - createToken
  - upsertPdpByAsin
  - getPdpByAsin
generated: '2026-08-02'
method: generated
source: https://docs.vizit.com/api
---

# Score an Amazon listing by ASIN

Use this when you have an Amazon ASIN and want Vizit's listing score without supplying
any images yourself. Vizit runs the whole pipeline server-side: ASIN lookup and
validation, category inference with an ICP check, retailer scrape, image ingest, and
scoring.

## Prerequisites

- A `client_id` / `client_secret` pair and an Auth0 `organization` ID, provisioned by
  Vizit for the environment you are calling. Credentials are environment-specific.
- The environment base URL: `https://ext.vizit.com` in production, or
  `https://dev1.ext.vizit.com` … `https://dev5.ext.vizit.com` for development.

## Step 1 — Get a Bearer token (`createToken`)

`POST /auth/token` — this is the only unauthenticated endpoint.

Body: `client_id`, `client_secret`, `audience` (always `https://ext.vizit.com`, in every
environment), `organization`, and `grant_type: "client_credentials"`.

The response carries `access_token`, `token_type` and `expires_in` (documented example:
86400 seconds). **Cache the token until it expires** — do not request one per call. On a
`401`, fetch a fresh token and retry once.

## Step 2 — Submit the ASIN (`upsertPdpByAsin`)

`PUT /v1/pdps/asin/{asin}` with `Authorization: Bearer <token>`.

- The ASIN must be 10 uppercase alphanumeric characters, or you get `400 INVALID_ASIN`.
- The body is optional. Send `{"region": "us"}` (or `uk`, `de`, `ca`, …) to choose the
  Amazon storefront; an omitted body is treated as `{"region": "us"}`. An unknown region
  returns `400 INVALID_REGION`.
- Send `X-Request-Id` so your logs correlate with Vizit's; it is echoed back.

Read the status code, not just the body:

- `202 Accepted` — a scrape was queued; `status` is `PROCESSING`.
- `200 OK` — the PDP was already terminal and nothing was triggered; `status` is
  `COMPLETED` or `ERROR`.

Either way the response returns `score_url`, a relative path that already carries the
`region` query parameter. Use it rather than rebuilding the URL.

This call is an **upsert**: submitting an ASIN that already exists in your organization
and region refreshes it, the same as the in-product refresh action.

## Step 3 — Poll for scores (`getPdpByAsin`)

`GET /v1/pdps/asin/{asin}?region=us` — or just follow `score_url`.

Poll until `status` reaches the terminal `COMPLETED` or `ERROR`. Expect tens of seconds
for a typical Amazon ASIN. During a refresh the **previous** scores stay visible
alongside `status: PROCESSING` and the prior `scored_at`, so you can keep displaying the
last known result while a new one computes — check `status` before treating scores as
current.

On `COMPLETED`, read:

- `pdp_score` — aggregate listing score, 0–100.
- `vizit_certified` — true when `pdp_score` is 80 or above.
- `carousel_score`, `hero`, `carousel_images[]` (each with `image_id`, `position`,
  `vizit_score`, `vizit_certified`, `classification`).
- `listing_score_at_ingest` and `score_change` — the baseline and the delta since ingest.
- The penalty flags `score_penalty`, `image_count_penalty`, `image_mix_penalty`,
  `image_order_penalty`, plus `high_scoring_asset_count`, `is_ideal_content_mix` and
  `images_matching_ideal_mix`.

Note the CSV export reports `image_count_penalty` / `image_mix_penalty` /
`image_order_penalty` as score **multipliers**, while this API reports them as
true/false flags.

## Errors to handle

Branch on the error code, never on the message.

| Code | Status | What to do |
|---|---|---|
| `INVALID_ASIN` | 400 | Fix the ASIN format before retrying. |
| `INVALID_REGION` | 400 | Use a supported Amazon storefront region. |
| `CATEGORY_NOT_IN_ORG_ICP` | 400 | The scraped category is outside your ICP — not retryable. |
| `NO_VALID_CATEGORY` | 400 | The retailer returned no usable category data. |
| `ASIN_NOT_FOUND_AT_RETAILER` | 404 | Amazon did not return the listing. |
| `PDP_NOT_FOUND` | 404 | Nothing ingested for this ASIN/region in your org. |
| `SCRAPE_UPSTREAM_ERROR` | 502 | Retry with backoff. |
| `SCRAPE_TIMEOUT` | 504 | Retry with backoff. |
| — | 429 | Honor `Retry-After`, then resume. |

Retry `429` and `5xx` with exponential backoff — start around 5s, double each retry, cap
around 60s. Do not retry other `4xx` until you have fixed the request.

## Rate limits

Per-organization limits are enforced at the API gateway; the exact tier is set when your
credentials are provisioned. Treat roughly a dozen submissions per minute per
organization as a safe baseline unless told otherwise.
