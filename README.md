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

| Sprint                                       | Endpoints | Status  |
| -------------------------------------------- | --------- | ------- |
| Faz 0 (initial skeleton)                     | 18        | Synced  |
| Sprint 1.4 pricing quote                     | +1        | Synced  |
| Sprint 1.5 admin removal (ADR-020)           | -30       | Synced  |
| Sprint 1.5 i18n response refactor (ADR-018)  | schema    | Synced  |
| Sprint 1.5 tenant-scoped currencies (ADR-019)| +1        | Synced  |
| Sprint 1.5 stop-sale check                   | +1        | Synced  |
| Sprint 1.5 reservations CRUD                 | +3        | Synced  |
| Sprint 1.5 multi-currency display            | schema    | Synced  |
| Sprint 2.1 branch concept                    | +1        | Synced  |
| Sprint 2.2 customer KYC + addresses/documents| schemas   | Synced  |
| Sprint 2.5 service + insurance + stop-sale   | schemas   | Synced  |
| Sprint 2.3 reservation core schemas + payments| +1       | Synced  |

### Sprint 1.5 architecture refactors

Three architectural decisions reshape the public contract in this
revision:

- **ADR-020 — admin endpoints removed**. The Backoffice connects
  directly to the database, so the 30 `/v1/admin/*` endpoints added
  in Sprint 1.4 have been removed. The corresponding `admin-*` tags
  are gone too.
- **ADR-018 — i18n response format**. Localized fields
  (`Vehicle.description`, `Vehicle.features`, `VehicleClass.name`,
  `VehicleClass.description`, `Location.name`, `Location.address`)
  are now emitted as a single localized string for the request
  locale (resolved from `Accept-Language`, falling back to
  `tenant.locale_default`). The `I18nString` schema has been deleted
  — translation maps are an internal Backoffice concern and are not
  exposed by the public API.
- **ADR-019 — tenant-scoped currencies**. The platform `Currency`
  schema now holds the ISO 4217 master record (`code`, `iso_name`,
  `decimal_places`, `default_symbol`). The new `TenantCurrency`
  schema is the per-tenant view returned by `GET /v1/currencies` —
  it carries the localized `name`, the effective `symbol`, the
  `is_default` flag, the cross-rate to the tenant default
  (`rate_to_default`) and the rate-source strategy
  (`rate_strategy`).

The Sprint 1.5 multi-currency fields (`display_currency`,
`display_total`, `display_rate`, `rate_source`, `rate_effective_at`)
on `PricingQuote` are now part of this revision — they ship together
with the live cross-rate engine and the new `TenantCurrency` model.

### Sprint 2.1 — Branch konsepti

Sprint 2.1 introduces the tenant **Branch** (sube) as a first-class
operating unit:

- **`Branch` schema** — public, B2B-visible representation with
  `id`, `code`, localized `name` / `address`, `phone`, `email`,
  `is_default`, `is_active`. Localized fields follow ADR-018
  (single string for the request locale).
- **`GET /branches`** — HMAC-authenticated B2B listing of the
  tenant's active branches; intended for partner storefront flows
  that need a "pickup branch" selector.
- **`branch_id` (nullable)** has been added to `Customer`,
  `CustomerInput`, `VehicleUnit`, `Location`, `Reservation` and
  `ReservationCreateRequest`. `null` means tenant-wide (HQ); a
  value pins ownership to a specific branch.
- **`BranchIdQuery` parameter** — common `?branch_id=` filter
  applied to `GET /locations` and `GET /reservations`. The POST
  `/search/availability` endpoint accepts the same filter inside
  `AvailabilityRequest.branch_id` (matching the existing
  body-filter pattern of `vehicle_class_id`).
- **`Branches` tag** added.

Branch CRUD is **not** in the public API — per ADR-005 / ADR-020
the Backoffice writes branches directly against the database.

Counts after Sprint 2.1: operations 20→21, schemas 29→30,
parameters 33→34, tags 10→11.

### Sprint 2.2 — Customer KYC + addresses + documents

Sprint 2.2 broadens the customer model with B2B / KYC fields and
introduces two embeddable relations (addresses and KYC
documents):

- **`CustomerAddress` / `CustomerAddressInput` schemas** — postal
  addresses with `type` (`home`, `work`, `billing`),
  `address_line1/2`, `city`, `district`, ISO 3166-1 alpha-2
  `country_code`, `postal_code` and an `is_default` flag (one
  default per `(customer_id, type)`).
- **`CustomerDocument` / `CustomerDocumentInput` schemas** — KYC
  documents (`identity`, `driver_license`, `passport`, `other`)
  whose binaries live in Cloudflare R2; only the object key plus
  metadata cross the API surface, with optional `expires_at` for
  documents that carry a validity end date.
- **`Customer` schema extension** — new fields `type`
  (`individual` / `corporate`), `tax_id`, `company_name`,
  `gender`, `license_class`, `license_country`, `source`, plus
  the embeddable `addresses` and `documents` arrays surfaced via
  the opt-in `?include=addresses,documents` query.
- **`CustomerInput` schema extension** — same KYC fields for the
  reservation-time upsert path, plus optional `addresses[]` and
  `documents[]` for first-onboarding bulk create. The `tckn`
  pattern is tightened to `^[1-9]\d{10}$` (first digit non-zero);
  the 10th/11th-digit checksum is enforced server-side, MerNIS /
  NVI live verification is deferred to Faz 2.
- **`ReservationCreateRequest.customer`** — now `oneOf
  CustomerInput | { id }`. The `{ id }` branch lets B2B partners
  link an existing tenant-scoped customer instead of re-supplying
  the full payload.

Customer / address / document CRUD is **not** in the public API
— per ADR-005 / ADR-020 the Backoffice writes these tables
directly. The schemas are exposed publicly only because they are
embeddable on the `Customer` aggregate and ship alongside
`CustomerInput` on `POST /reservations`.

Counts after Sprint 2.2: operations 21 (unchanged), schemas
30→34 (`CustomerAddress`, `CustomerAddressInput`,
`CustomerDocument`, `CustomerDocumentInput`), parameters 34
(unchanged), tags 11 (unchanged).

### Sprint 2.5 — Service + insurance + stop-sale transparency

Sprint 2.5 closes the off-fleet visibility gap by exposing the
read-only side of the new service / insurance tables and adding
an opt-in transparency field on availability search:

- **`VehicleService` schema** — workshop / muayene / repair
  records (`maintenance`, `repair`, `inspection`, `insurance`)
  with `[starts_at, ends_at]`, optional `km_at_service`, `cost`,
  `currency_code`, `vendor` and free-text `notes`. Mirrors the
  new `vehicle_services` table; treated by the availability
  engine as a hard block on the unit (ADR-021).
- **`InsuranceCompany` schema** — tenant-scoped insurer
  directory entry (`name`, `contact_phone`, `contact_email`,
  `is_active`).
- **`VehicleInsurance` schema** — per-plate policy term with
  `policy_no`, `coverage_type` (`kasko` / `trafik` / `imm`),
  `[starts_at, expires_at]`, optional `premium`,
  `currency_code`, `deductible`, plus an embeddable
  `insurance_company` relation (opt-in via `?include=`).
- **`AvailabilityOffer.unavailability_reasons`** — opt-in
  informational array (one entry per reservation / stop-sale /
  service window intersecting the request) so partner
  storefronts can debug why a template's `units_available`
  dropped. Empty / null on fully-available templates; never
  carries PII (reservation rows expose only the conflict
  window).
- **`StopSale.branch_id`** — nullable branch scoping for
  stop-sale records, matching the Sprint 2.1 branch
  propagation.

Service / insurance / stop-sale CRUD is **not** in the public
API — per ADR-005 / ADR-020 the Backoffice writes these tables
directly. The schemas are exposed publicly only because they
surface in `unavailability_reasons` and on the read paths.

Counts after Sprint 2.5: operations 21 (unchanged), schemas
34→37 (`VehicleService`, `InsuranceCompany`,
`VehicleInsurance`), parameters 34 (unchanged), tags 11
(unchanged).

### Sprint 2.3 — Reservation core (extras + drivers + payments + status history)

Sprint 2.3 fleshes out the Reservation aggregate so the
Backoffice and B2B partners can drive the full reservation
lifecycle through the public surface:

- **`ReservationExtra` schema** — line-item extra (child seat,
  GPS, additional driver…) with snapshot `name` / `unit_price`
  / `currency_code`, `quantity`, server-computed `total`, and
  optional FK back to the catalogue (`extra_id`, nullable so a
  deleted catalogue row leaves the line intact).
- **`ReservationDriver` schema** — driver assignment with
  `customer_id`, `is_primary`, optional `license_verified_at`
  and an embeddable `customer` relation
  (`?include=drivers.customer`).
- **`ReservationPayment` schema** — payment line with `amount`,
  `currency_code`, `method` (`cash` / `card` / `bank_transfer`
  / `installment`), `status` (`pending` / `captured` /
  `refunded` / `failed`), `transaction_ref`, `paid_at`.
- **`ReservationStatusHistoryEntry` schema** — append-only
  status transition log (`from_status` → `to_status`,
  `changed_by_user_id`, `reason`, `changed_at`); the creation
  event has `from_status = null`.
- **`Reservation` schema extension** — operational columns
  (`type` `rentacar` / `transfer`, `agent_user_id`, `source`
  `online` / `backoffice` / `phone` / `b2b`, `extras_total`,
  `drop_fee_amount`, `taxes`, `discount_amount`,
  `payment_method`, `contract_no`, `internal_notes`,
  `custom_fields`) plus the four embeddable arrays (`extras`,
  `drivers`, `payments`, `status_history`) opt-in via
  `?include=`.
- **`ReservationCreateRequest` extension** — bulk
  `extras[]` (extra_id + quantity) and `drivers[]` (customer_id
  + is_primary) on create, plus `payment_method`,
  `discount_amount` and `custom_fields`.
- **`POST /reservations/{reservation_number}/payments`** —
  HMAC-authenticated B2B partner notification flow ("we
  received the customer's payment, please record it"). The
  Backoffice still writes payments DB-direct per ADR-005 /
  ADR-020. Idempotent via `Idempotency-Key`. The cancel
  endpoint shipped in Sprint 1.5 already covers the partner
  cancel use case (`PATCH /reservations/{reservation_number}/cancel`).

Reservation extra / driver / payment / status-history CRUD on
existing reservations remains a Backoffice DB-direct concern
per ADR-005 / ADR-020. The schemas are exposed publicly only
because they are embeddable on the `Reservation` aggregate
and / or surface on the new payment-notification endpoint.

Counts after Sprint 2.3: operations 21→22 (`POST
/reservations/{reservation_number}/payments`; cancel was
already shipped in Sprint 1.5), schemas 37→41
(`ReservationExtra`, `ReservationDriver`,
`ReservationPayment`, `ReservationStatusHistoryEntry`),
parameters 34 (unchanged), tags 11 (unchanged).
