# Vizit

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

Vizit (Vizit Labs, Inc.) is a Boston-based Visual AI company whose platform predicts,
measures, optimizes and monitors the effectiveness of ecommerce visual content. Its
patented Audience Lens technology scores product imagery against deep-learning models
trained to simulate the visual preferences of specific consumer audiences.

- Website: https://www.vizit.com/
- Developer docs: https://docs.vizit.com/
- API reference: https://docs.vizit.com/api
- Status: https://status.vizit.com/
- Help center: https://help.vizit.com/

## Vizit Public API

Machine-to-machine REST API at `https://ext.vizit.com` (plus `dev1`–`dev5` development
environments), authenticated with OAuth 2.0 client credentials via Auth0 and a 24-hour
Bearer JWT. Sixteen published operations across six capabilities: authentication,
product details (PDP ingest by Amazon ASIN or by caller id), images (standalone scoring
with GS1 hero sub-scores), Spark Ideas, Spark Images, and bulk CSV exports.

## Artifacts

| Artifact | Path |
|---|---|
| Authentication profile | `authentication/vizit-authentication.yml` |
| API conventions (incl. idempotency) | `conventions/vizit-conventions.yml` |
| Error catalog | `errors/vizit-error-codes.yml` |
| Rate limits | `rate-limits/vizit-rate-limits.yml` |
| Lifecycle + status page | `lifecycle/vizit-lifecycle.yml` |
| Sandbox / environments | `sandbox/vizit-sandbox.yml` |
| Conformance | `conformance/vizit-conformance.yml` |
| Data model | `data-model/vizit-data-model.yml` |
| MCP (candidate) | `mcp/vizit-mcp.yml` |
| Agent skills | `skills/_index.yml` |
| llms.txt (verbatim) | `llms/vizit-llms.txt` |
| Well-known probe record | `well-known/vizit-well-known.yml` |
| Domain security | `security/vizit-domain-security.yml` |

## Known gaps

- **No downloadable OpenAPI.** The docs state the reference is generated from the Vizit
  Public API OpenAPI specification, but no spec document is retrievable:
  `/openapi.json`, `/openapi.yaml`, `/swagger.json`, `/docs` and `/redoc` return 401 on
  `api.vizit.com` and 403 on `ext.vizit.com`, and 404 on `docs.vizit.com`. Publishing the
  spec at a public URL is the single highest-value fix available to Vizit.
- No `/.well-known/` surface on any host — no `security.txt`, no OAuth/OIDC discovery
  document, no API catalog, no A2A agent card.
- No first-party SDKs or CLI on npm, PyPI or any other registry; no GitHub organization.
- No dated changelog, no deprecation/sunset policy, no published SLA.
- No trust center or named compliance certifications; errors are a flat proprietary JSON
  envelope rather than RFC 9457.
