---
name: beacon-order-materials
description: >-
  Search the Beacon/QXO roofing catalog, get account-specific pricing, build a cart, validate and
  review it, then submit a material order against a contractor job. Use this when a contractor or an
  estimating/CRM system needs to buy roofing materials from Beacon Roofing Supply (QXO).
api: Beacon Rest Services (Beacon PRO+)
provider: beacon-roofing-supply
base_url: https://beaconproplus.com/v2/rest/com/becn
spec: openapi/beacon-roofing-supply-all-api-openapi.yml
generated: '2026-09-04'
method: generated
source: >-
  Grounded in operationIds verified present in openapi/beacon-roofing-supply-all-api-openapi.yml on
  2026-09-04, plus the conventions and error rules captured in
  conventions/beacon-roofing-supply-conventions.yml and errors/beacon-roofing-supply-error-codes.yml
operations:
  - V1Controller_login
  - V1Controller_accounts
  - V1Controller_switchAccount
  - V1Controller_branchlist
  - V3Controller_itemlist
  - V3Controller_itemDetails
  - V2Controller_typeAhead
  - V2Controller_getProductVariation
  - V2Controller_pricing
  - V2Controller_getProductBranchOrRegionAvailability
  - V2Controller_getSKUBranchOrRegionAvailability
  - V2Controller_addMultipleItemsToOrder
  - V2Controller_updateCart
  - V2Controller_cartItems
  - V2Controller_removeItemFromCart
  - V2Controller_clearCart
  - V2Controller_updateCurrentOrderJobNumber
  - V2Controller_addOrderShippingInfo
  - V2Controller_validateOrderByLocation
  - V2Controller_getCurrentOrderReview
  - V2Controller_proceedToCheckout
  - V2Controller_submitOrder
  - V2Controller_submitCurrentOrder
  - V2Controller_getSubmitOrderResult
---

# Order roofing materials from Beacon (QXO)

## STOP — read this before you write any code that calls submitOrder

Two properties of this API decide how you must build:

1. **There is no idempotency mechanism.** No `Idempotency-Key` header, no client-supplied dedupe key
   on any write. If `POST /submitOrder` times out and you retry, you may place a second real order.
2. **There is no cancellation operation.** Nothing in the 424 published operations cancels, voids or
   amends a submitted order. Beacon's own documentation routes cancellation through a branch
   representative, and documents that `checkForAvailability` should always be `"No"` precisely so a
   human at the branch receives the order.

So: **submitting is one-way and un-retryable.** Require explicit human confirmation before the submit
step, and on any ambiguous failure reconcile with `V2Controller_orderhistory` before doing anything
else. Never auto-retry a submit.

## 1. Authenticate

V2 uses an OAuth 2.0 bearer token: `Authorization: Bearer <access_token>`.

Refresh with `POST https://beaconproplus.com/rest/model/REST/oauth/token`:

```json
{ "grant_type": "refresh_token", "refresh_token": "<token>", "client_id": "<client_id>", "scopes": "all" }
```

`scopes` here is a refresh selector, not a permission: empty refreshes only the access token, `all`
or `refresh_token` refreshes both.

The **initial** grant is not published. Credentials come from partner onboarding at
https://go.qxo.com/qxoapi. If you only have a PRO+ web login, V1 `V1Controller_login` establishes a
cookie session (JSESSIONID / DYN_USER_ID / DYN_USER_CONFIRM) instead — 1 hour, or 7 days with the
RememberMe flag.

## 2. Pick the account and branch

- `V1Controller_accounts` — the accounts this profile may act for.
- `V1Controller_switchAccount` — set the working account. Message code **5006** ("Need change
  account") means you are operating against the wrong one.
- `V1Controller_branchlist` — the branches serving the account. `branchNumber` drives regional
  catalog, availability and delivery.

Confirm what you are allowed to do: `V2Controller_getCurrentUserPermission`. Permission is enforced
server-side by permission template; there is no scope string to inspect. A 403 with message code
**2006** means permission denied, and eleven operations are restricted to master-admin/admin users.

## 3. Find products

- `V2Controller_typeAhead` for search-as-you-type.
- `V3Controller_itemlist` for faceted catalog search. Useful flags: `showPricing`, `showFacets`,
  `showSkuList`, `showItemAvailable`, `showItemVariations`, `enableAutoCorrection`, `enableDidYouMean`.
- `V3Controller_itemDetails` for full product detail.
- `V2Controller_getProductVariation` / `getMultipleProductVariation` to resolve a product to a
  purchasable SKU (colour, size, unit of measure).

Page with `pageNo` / `pageSize`. Responses that use the shared `BCPaginationDataObj` envelope give you
`next`, `previous`, `pageSize`, `currentPage`, `totalCount`, `results`. Watch out: on 7 operations
`pageNo`/`pageSize` are **path** parameters rather than query parameters.

Do **not** rebuild the whole catalog by paging `itemlist` — use the bulk skill
(`beacon-sync-catalog`) instead. The license terms prohibit usage that "exceeds reasonable request
volume" without defining a threshold, and bulk extract endpoints exist for exactly this.

## 4. Price and check availability

- `V2Controller_pricing` — account-specific pricing. Prices are negotiated per account, so never
  cache one account's price for another.
- `V2Controller_getProductBranchOrRegionAvailability` and `V2Controller_getSKUBranchOrRegionAvailability`.

Message code **3002** / **3006** mean the product or SKU is not available in the current profile's
region. That is a catalog-scope error, not a stock-out.

## 5. Build the cart

- `V2Controller_addMultipleItemsToOrder` to add a list of items.
- `V2Controller_updateCart` to change quantities.
- `V2Controller_cartItems` to read the current cart.
- `V2Controller_removeItemFromCart` (one line) or `V2Controller_clearCart` (everything) to undo.

Everything in this step is reversible. Attribute the spend to a project with
`V2Controller_updateCurrentOrderJobNumber` — `jobNumber` is the field a job-costing or ERP
integration hangs on, and it appears on 48 schema properties across the contract.

Set delivery with `V2Controller_addOrderShippingInfo`. Keep `checkForAvailability` at `"No"` unless
you genuinely want the order cancelled when stock is short — Beacon documents `"Yes"` as
auto-cancelling.

## 6. Rehearse before you commit

This is the part most integrations skip, and it is the only safety net the API gives you:

- `V2Controller_validateOrderByLocation` — validate the order against branch/region rules.
- `V2Controller_getCurrentOrderReview` — the full review payload as PRO+ would show it.
- `V2Controller_proceedToCheckout` — the checkout step.

Show the review output to a human. Get confirmation. Then, and only then, submit.

## 7. Submit — once

- `V2Controller_submitCurrentOrder` submits the working cart.
- `V2Controller_submitOrder` submits a constructed order payload.
- `V2Controller_getSubmitOrderResult` retrieves the result of a submission.

Include your own `atgUUID` (a GUID you generate) on the order. Beacon echoes it back. It is **not**
documented as a deduplication key and will not stop a double submission — but it is the only
correlation handle you own, and it is what makes the reconciliation in the next paragraph possible.

**On timeout or any 5xx from a submit:** do not retry. Call `V2Controller_orderhistory` filtered by
`jobNumber` and your recent window, and match on your `atgUUID` / PO number. If the order is there,
you are done. If it is not, escalate to a human rather than resubmitting.

Failure codes to expect: **5001** submit order failed, **5002** submit failed with an integration
exception, **5010** no current order, **4003** ATG order version invalid, **4002** Mincron error
(commercial/ERP-backed accounts).

## Error handling

Every response is `{ success, messages: [{key, code, type, value}], result }` — except 27 named V2
operations that return the raw payload with no envelope (listed in
`errors/beacon-roofing-supply-error-codes.yml`), which includes `/submitOrder` itself.

Status codes are not conventional. Branch on them like this:

| Status | Meaning here | Action |
|---|---|---|
| 400 | Bad request, usually an invalid `accountId` | Fix the account, do not retry blind |
| 401 | Token invalid, or (V1) not logged in | Re-authenticate |
| **419** | Bearer token **expired** — not an IANA status code | Refresh the token and retry |
| 403 | Permission denied (code 2006) | Stop. A retry will not help |
| **405** | Wrong method **or** the operation is not enabled for your site | Check enablement before changing the request |
| 500 | Internal exception | Back off with jitter; for a submit, reconcile instead of retrying |

Every error response carries an undocumented top-level `traceId`. Log it — it is what Beacon support
will ask for.

There is no rate-limit signal at all: no 429, no `RateLimit-*`, no `Retry-After`. Rate your own
traffic conservatively and ask for your limit during onboarding.
