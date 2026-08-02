# Vizit

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
