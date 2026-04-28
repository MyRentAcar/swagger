# Rent A Car SaaS — OpenAPI Spec

This repository holds the canonical OpenAPI 3.1 contract for the multi-tenant
rent-a-car SaaS platform. It is the integration surface shared by:

- the Laravel 13 monolith API (annotation-driven, generated via L5-Swagger),
- the Next.js 15 storefront and Cloudflare Worker proxy,
- the iOS (Swift) and Android (Kotlin) mobile apps,
- B2B agency partners (Passport `client_credentials` clients).

Architectural background lives in the platform `docs/` tree:

- [`docs/02-architecture.md`](../docs/02-architecture.md) — overall system & API style.
- [`docs/08-api-security.md`](../docs/08-api-security.md) — HMAC, rate limit, Sanctum, Passport.
- [`docs/15-b2b-agency-design.md`](../docs/15-b2b-agency-design.md) — B2B agency channel.
- [`docs/16-multi-currency-design.md`](../docs/16-multi-currency-design.md) — multi-currency model.
- [`docs/adr/ADR-009-api-documentation.md`](../docs/adr/ADR-009-api-documentation.md) — why this repo exists.

## Folder layout

```
openapi/
├── openapi.yaml                    # Root spec — refs everything below
├── components/
│   ├── parameters/                 # Common headers + query params
│   ├── responses/                  # Common error envelopes (RFC 7807)
│   ├── schemas/                    # Domain models (one file per type)
│   └── securitySchemes/            # sanctumBearer / hmacAuth / passportOAuth
└── paths/                          # Endpoint groups (one file per group)
    ├── auth.yaml
    ├── b2b-agency.yaml
    ├── currencies.yaml
    ├── locations.yaml
    ├── reservations.yaml
    ├── search.yaml
    ├── stop-sale.yaml
    ├── tenants.yaml
    └── vehicles.yaml
.github/workflows/openapi-validate.yml
.redocly.yaml                       # Linting style guide
package.json
```

The root `openapi.yaml` references every component and path file via JSON
pointers. Editors and IDEs that understand `$ref` (Stoplight, Insomnia,
Redocly preview) will navigate the tree natively.

## Multi-tenancy and platform conventions

Every operation requires the application-identity (HMAC) headers and the
`X-Tenant-ID` header:

- `X-App-Id`, `X-Timestamp`, `X-Nonce`, `X-Signature` — see
  [`docs/08-api-security.md` §6](../docs/08-api-security.md).
- `X-Tenant-ID` — resolved by the `IdentifyTenant` middleware
  ([`docs/02-architecture.md` §2.2](../docs/02-architecture.md)).
- `X-Display-Currency` — optional, controls display-layer currency
  conversion (storage stays in tenant default; see ADR-012).
- `Accept-Language` — `tr`, `en`, `de`, `ru`, `ar`.
- `Idempotency-Key` — required for `POST /reservations` and
  `POST /agency/reservations`.

Errors follow [RFC 7807](https://www.rfc-editor.org/rfc/rfc7807)
(`application/problem+json`). Validation errors carry an `errors` map keyed
by request field (Laravel FormRequest convention).

## Quickstart

```bash
# Node 20+
npm ci

# Lint
npm run lint

# Validate (swagger-cli)
npm run validate

# Bundle into a single file
npm run bundle              # → dist/openapi.bundled.yaml
npm run bundle:json         # → dist/openapi.bundled.json

# Local docs preview (http://localhost:8080)
npm run preview

# Repo statistics (paths, schemas, …)
npm run stats
```

`npm run build` chains lint + bundle + bundle:json — it is the same set of
commands the CI workflow runs.

## Adding a new endpoint

1. **Decide the group**. Files in `openapi/paths/` are organized by domain
   (auth, vehicles, reservations…). Add a new file only for a new domain
   group.
2. **Reuse parameters and schemas**. The HMAC + tenant + display currency +
   language headers live in `openapi/components/parameters/` and are
   referenced by every operation. Reuse them — do not re-declare.
3. **Add a path entry to `openapi.yaml`** under `paths:` using a `$ref` to
   the YAML key in your group file (mirror the existing pattern).
4. **Document the operation**:
   - Stable `operationId` (`<group>.<verb>`, dot-separated, kebab-safe).
   - `summary` (≤60 chars), `description` (multi-line, references docs).
   - `tags` matching one of the tags defined in `openapi.yaml`.
   - `security` block (HMAC always; add `sanctumBearer` or `passportOAuth`
     as needed).
   - At least the `2xx` plus `401`/`422`/`429` responses.
5. **Run `npm run lint`** locally before opening a PR. CI runs the same
   command on every push and PR (see `.github/workflows/openapi-validate.yml`).
6. **PRs that change the public contract** should be tagged `[api-spec]`
   and trigger the `oasdiff` job in CI; an unintended breaking change
   blocks merge per [ADR-009](../docs/adr/ADR-009-api-documentation.md).

## CI / Auto-sync from the API repo

This repo is the **publish target**, not the source of truth. The
end-to-end workflow described in
[ADR-009](../docs/adr/ADR-009-api-documentation.md) is:

1. Engineers add OpenAPI annotations to Laravel controllers in the API
   repo (`darkaonline/l5-swagger`).
2. On merge to `main`, GitHub Actions in the API repo runs
   `php artisan l5-swagger:generate`, opens a PR against this repo with the
   refreshed `openapi/openapi.yaml` (or `dist/openapi-v1.yaml` for the
   bundled artifact).
3. The PR triggers `openapi-validate.yml` here:
   - Redocly lint (style guide).
   - `swagger-cli validate` (structural).
   - `redocly bundle` (single-file artifacts).
   - `oasdiff` breaking-change detector vs `main`.
4. On merge, downstream consumers update:
   - **Swagger UI** at `docs.example.com/api/v1` — Cloudflare Pages reads
     `dist/openapi.bundled.yaml`.
   - **Postman collection** generated by `openapi-to-postmanv2` and synced
     to the team workspace via the Postman API. Custom test scripts
     (HMAC pre-request, auth flow) live under `postman/test-scripts/`
     (added in a follow-up phase).
   - **Mobile / frontend SDKs** consume the bundled YAML directly or via
     `openapi-generator` in their own pipelines.

## Postman collection export (manual fallback)

Until the automated Postman sync is wired, partners can import the
bundled spec manually:

```bash
npm run bundle
# Postman → File → Import → dist/openapi.bundled.yaml
# (Postman auto-detects OpenAPI 3.1 and offers to generate the collection.)
```

The HMAC pre-request script template lives in `docs/08-api-security.md`
§6.6.1 (Cloudflare Worker version) and can be adapted to Postman's
`pm.sendRequest` runtime.

## Data residency

Production traffic is served from `https://api.siteadi.com` only, hosted
in TR per [ADR-014](../docs/adr/ADR-014-data-residency-tr.md). The
staging server in `servers:` is for non-production traffic and is not
publicly advertised.

## Versioning

We use path versioning (`/v1`, `/v2`). During the 6-month deprecation
overlap defined in ADR-009, an `X-Api-Version` request header may pin a
client to a specific major. Each major lives in its own file pair
(future: `openapi/openapi-v2.yaml`).

## Sync status

This spec lags the implementation by one PR per sprint. The table below
tracks which sprint's endpoints are reflected in `openapi/openapi.yaml`.

| Sprint                              | Endpoints | Status   |
| ----------------------------------- | --------- | -------- |
| Faz 0 (initial skeleton)            | 18        | Synced   |
| Sprint 1.4 admin CRUD (6 resources) | +30       | Synced   |
| Sprint 1.4 pricing quote            | +1        | Synced   |
| Sprint 1.5 stop-sale check          | +1        | Pending  |
| Sprint 1.5 reservations CRUD        | +5        | Pending  |
| Sprint 1.5 currencies               | +2        | Pending  |
| Sprint 1.5 multi-currency display   | schema    | Pending  |

Sprint 1.5 endpoints land in a follow-up sync once the parallel
`monolithapi` sprint merges. The Sprint 1.5 multi-currency fields
(`display_currency`, `display_total`, `display_rate`, `rate_source`,
`rate_effective_at`) on `PricingQuote` are intentionally omitted from
this revision — they ship together with the live cross-rate engine.
