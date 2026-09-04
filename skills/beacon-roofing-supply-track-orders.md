---
name: beacon-track-orders
description: >-
  Pull Beacon/QXO order history, order detail, delivery tracking, invoices and manufacturer rebates
  into a job-costing ledger, CRM or project-management tool. Read-only — no write operation is used.
api: Beacon Rest Services (Beacon PRO+) V2
provider: beacon-roofing-supply
base_url: https://beaconproplus.com/v2/rest/com/becn
spec: openapi/beacon-roofing-supply-v2-openapi.yml
generated: '2026-09-04'
method: generated
source: >-
  Grounded in operationIds and tags verified present in
  openapi/beacon-roofing-supply-all-api-openapi.yml and openapi/beacon-roofing-supply-v2-openapi.yml
  on 2026-09-04
operations:
  - V2Controller_orderhistory
  - V2Controller_orderhistory_v2
  - V2Controller_orderdetail
  - V2Controller_orderSummary
  - V2Controller_getDTOrderDetail
  - V2Controller_downloadOrderDetailAsPDF
  - V2Controller_downloadOrderDocument
  - V1Controller_jobs
  - V2Controller_getCurrentUserLastSelectedJobInfo
  - V2Controller_rebateLanding
  - V2Controller_getRebateRedeemedSummaryItems
  - V2Controller_getRebateRedeemedItemDetail
  - V2Controller_getRebateForm
  - V2Controller_updateOrderAlert
---

# Track Beacon (QXO) orders, deliveries, invoices and rebates

This is the read side of the Beacon API and the safest surface to automate: every operation below is
a GET or a read-shaped POST, so none of the idempotency and no-cancellation hazards that govern
`/submitOrder` apply here.

## Orders

- `V2Controller_orderhistory` and `V2Controller_orderhistory_v2` — paged history. Filter by
  `accountId`, `jobNumber`, `searchStartDate` / `searchEndDate`, `searchBy` / `searchTerm`, and
  `queryCompany` (added across the order endpoints in release `24PI-3-Sprint-4`).
- `V2Controller_orderdetail` — full detail for one order.
- `V2Controller_orderSummary` — summary view.
- `V2Controller_downloadOrderDetailAsPDF` and `V2Controller_downloadOrderDocument` — documents for
  the order, if you need the paper trail attached to the job.

Order identifiers: `orderId` / `orderNumber`, plus `atgOrderId` (the number as it appears in PRO+,
format `KP32234`) and `atgUUID` (a caller-supplied GUID Beacon echoes back). If your system set an
`atgUUID` at submission, that is your join key.

## Attribute spend to a job

`jobNumber` is the field that makes this data useful. It appears on 48 schema properties and 20
operation parameters and is what turns an order history into a job-cost ledger.

- `V1Controller_jobs` — the contractor's jobs.
- `V2Controller_getCurrentUserLastSelectedJobInfo` — the job currently in context.
- Filter `orderhistory` by `jobNumber` to get per-project material spend.

## Deliveries

- `V2Controller_getDTOrderDetail` — delivery-tracking order detail.
- The *Delivery Tracking Service* tag in the V2 document carries the rest of the surface.
- `saveDeliverySetting` / `getDeliverySetting` control notification preferences; release
  `24PI-2-Sprint-5` added `enRoute` and `arrived` elements to both.
- `V2Controller_updateOrderAlert` manages order alerts.

## Invoices

Invoice history is published under the *Bill Trust Services* tag in the V2 document
(`/invoiceHistory`, and `/invoice` on the internal surface). QXO markets this as its **Invoice API** —
"invoices, invoiced data, payment status and other information used in financials"
(https://www.qxo.com/customapi). Note these operations have **no operationId** in the V2 document;
only the combined `all_api` document assigns operationIds, and the invoice paths are not in it.
Bind to document + path, not to an identifier.

## Rebates

Manufacturer rebates are real money for a roofing contractor and are frequently left uncollected:

- `V2Controller_rebateLanding` — the rebate landing payload.
- `V2Controller_getRebateRedeemedSummaryItems` — what has been redeemed.
- `V2Controller_getRebateRedeemedItemDetail` — line-level redemption detail.
- `V2Controller_getRebateForm` — the submission form (`V2Controller_submitRebate` writes, and is out
  of scope for this read-only skill).

## Polling, not webhooks

**Beacon publishes no webhooks and no AsyncAPI.** There is no event delivery of any kind. Order
status, delivery status and notifications are all **polled**:

- `GET /getNotifications` returns user notifications; `/deleteNotifications` and
  `/deleteAllNotifications` clear them.
- `POST /quickMeasure/callback` is Beacon *receiving* a report from GAF QuickMeasure — it is not
  Beacon emitting an event to you.

Design for polling. There is no rate-limit signal published (no 429, no `RateLimit-*`, no
`Retry-After`), and the license terms prohibit "excessive or abusive usage" without naming a number,
so choose a conservative interval — hourly for open orders, daily for closed ones — and negotiate
your actual limit during onboarding at https://go.qxo.com/qxoapi.

## Pagination

`pageNo` / `pageSize` (and `orderBy` on 10 operations). Where an operation returns the shared
`BCPaginationDataObj` envelope you get `next`, `previous`, `pageSize`, `currentPage`, `totalCount`,
`results`. On 7 operations `pageNo`/`pageSize` are path parameters instead of query parameters — check
the path before assuming.

## Errors

Envelope: `{ success, messages: [{key, code, type, value}], result }`. Watch for **419** (bearer token
expired — refresh and retry; it is not an IANA status code), **405** (wrong method *or* not enabled
for your site), **5004** ("No result is returned to order history" — an empty result, not a failure),
**5003** ("Query order detail not match with current") and **5007** ("Account no longer available for
current user", which usually means you need `V1Controller_switchAccount`). Log the `traceId` on every
error response. Full registry: `errors/beacon-roofing-supply-error-codes.yml`.
