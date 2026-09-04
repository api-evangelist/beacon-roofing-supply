---
name: beacon-sync-catalog
description: >-
  Bulk-extract the Beacon/QXO product catalog, SKU data, branch list and product availability into an
  ERP, PIM or estimating database, and map Beacon products to Mincron ERP identifiers. Use this
  instead of paging the search endpoints when you need the whole catalog.
api: Beacon Rest Services (Beacon PRO+) V3
provider: beacon-roofing-supply
base_url: https://beaconproplus.com/v3/rest/com/becn
spec: openapi/beacon-roofing-supply-v3-openapi.yml
generated: '2026-09-04'
method: generated
source: >-
  Grounded in paths and operationIds verified present in openapi/beacon-roofing-supply-v3-openapi.yml
  and openapi/beacon-roofing-supply-all-api-openapi.yml on 2026-09-04
operations:
  - V3Controller_downloadCatalogItemData
  - V3Controller_itemlist
  - V3Controller_itemDetails
  - V3Controller_mincronMapping
  - V1Controller_branchlist
  - V2Controller_categories
  - V2Controller_getGenericBrands
---

# Bulk catalog sync from Beacon (QXO) V3

V3 is Beacon's integration surface. It is small — eight published operations — and four of them exist
specifically so an ERP does not have to crawl the storefront. Use them.

## Authentication

Bearer token: `Authorization: Bearer <access_token>`. V3's securityScheme is
`bearerAuth {type: http, scheme: bearer, bearerFormat: token}`. Despite being titled "Public", every
V3 operation requires a token — probed live 2026-09-04, `GET /v3/rest/com/becn/branchData` returns
`401 {"success":false,...,"value":"Please provide token"}`.

## The bulk extracts

| Path | What it returns |
|---|---|
| `GET /downloadCatalogItemData` | Catalog item data including branches, branch SKUs and SKUs — the widest single extract (`V3Controller_downloadCatalogItemData`) |
| `GET /skuData` | SKU-level catalog extract |
| `GET /branchData` | Branch catalog extract |
| `GET /productAvailabilityData` | Product availability extract |

These four are the intended catalog-sync path. Reconstructing the catalog by paging
`V3Controller_itemlist` is the access pattern most likely to read as "excessive or abusive usage"
under section 3.1(f) of the published API license terms
(https://www.qxo.com/integrations/api-license-terms), which prohibits it without naming a number.

## Reference data to pull alongside it

- `V1Controller_branchlist` — the branches serving the account. `branchNumber` (e.g. `"800"`) keys
  availability and delivery; it is assigned from customer location.
- `V2Controller_categories` — the product hierarchy. Beacon uses its own codes, e.g.
  `US_MAIN_CAT_RESIRFNG_ASPHALT`, `US_MAIN_CAT_COMRFNG_TPOVA`, `US_MAIN_CAT_TRIBUILT_POLYISO`.
  There is **no GTIN/GS1 or UNSPSC mapping published** — if your PIM keys on GS1 identifiers you must
  build and maintain the crosswalk yourself.
- `V2Controller_getGenericBrands` — brand list.
- `V3Controller_itemDetails` — enrich a specific item after the bulk pass.

## Mincron ERP mapping

`GET /mincronMapping` (`V3Controller_mincronMapping`, tagged *Integration Services*) returns Mincron
mapping products. Mincron is Beacon's back-office ERP for commercial accounts and it surfaces
throughout the contract — `mincronId` fields, `POST /getMincronQuoteDetail`, and message code **4002**
("Mincron error"). If you are integrating a commercial account, pull this mapping first; without it,
quote and order identifiers will not reconcile against the ERP side.

Oracle ATG is the other back-office system that leaks into the contract — `atgOrderId` (format
`KP32234`), `atgUUID`, `GET /getAtgQuoteDetail`, and message code **4003**.

## Scheduling and freshness

- Prices are **account-specific**. A catalog extract is not a price list; call
  `V2Controller_pricing` per account. Never cache one account's price against another.
- Availability is **per branch and per region**. Message codes **3002** and **3006** mean a product
  or SKU is out of the current profile's region entirely, which is different from being out of stock.
- No cache headers or freshness guidance are published, and live API responses set
  `cache-control: no-cache, no-store, must-revalidate, max-age=0`. Pick your own cadence; nightly is
  the conventional choice for distribution catalogs.
- No rate limit, window or `Retry-After` is published anywhere. Serialize the bulk calls, back off on
  500, and ask Beacon for your limit at onboarding.

## Multi-site

Most operations accept `apiSiteId` (e.g. `"DEN"`). Resolution order is: query string, then a **root**
element of the JSON body (a nested one is ignored), then the site bound to your token's `client_id`.
Omitting it does not mean "no site" — it means the site your credentials are bound to.

## Errors

Same envelope and status conventions as the ordering skill: 419 = token expired (refresh and retry),
405 = wrong method *or* operation not enabled for your site, and every error carries a `traceId` worth
logging. Full registry: `errors/beacon-roofing-supply-error-codes.yml`.
