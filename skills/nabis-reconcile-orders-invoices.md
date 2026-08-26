---
name: nabis-reconcile-orders-invoices
description: Reconcile Nabis wholesale orders against their invoices and retailer credit standing — pull orders for a date window, join them to invoices and retailers, and read the credit and payment fields correctly.
api: Nabis Platform API v2
base_url: https://platform-api.nabis.pro
generated: '2026-08-26'
method: generated
source: openapi/nabis-platform-api-v2-openapi.yml + https://developers.nabis.com/v2/docs/overview/dates-and-times
operations:
  - OrderController_getOrders
  - OrderController_getOrder
  - InvoiceController_getInvoices
  - RetailerController_getRetailers
  - RetailerController_getRetailer
  - NabisDaysOffController_getNabisDaysOff
  - NYOrderController_getNyOrders
  - NYOrderController_getNyOrder
  - NYInvoiceController_getNyInvoices
  - NYRetailerController_getNyRetailers
---

# Reconcile Nabis orders, invoices and retailer credit

Authenticate every call with `x-nabis-access-token`. Everything here is read-only; you cannot
create, amend, cancel or pay anything through this API — those actions live in the Nabis web
application at app.nabis.com.

## Steps

1. **Pull the order window.** `OrderController_getOrders` — `GET /v2/order?page=0&limit=500`.
   Narrow it with the date pairs, all `YYYY-MM-DD` and UTC:
   `startCreatedAt`/`endCreatedAt`, `startDeliveryDate`/`endDeliveryDate`,
   `startUpdatedAt`/`endUpdatedAt`, `startLastNonPaymentDate`/`endLastNonPaymentDate`.
   Put the **same date in both** ends of a pair to get a single day.
   Further filters: `paymentStatus`, `paymentTerms`, `action`, `status`, `referrer`, `warehouse`,
   `lastNonPaymentReason`.
2. **Expect one row per line item, not per order.** Line items are flattened into the order response
   as `lineItem*`-prefixed columns (`lineItemId`, `lineItemSubtotal`,
   `lineItemSubtotalAfterDiscount`, `lineItemDiscount`, `lineItemPricePerUnitAfterDiscount`,
   `lineItemWholesaleValue`, `lineItemNotes`, `lineItemUpdatedAt`). An order with 6 lines is 6 rows
   sharing one `id`. Group by `id`, and use `orderTsPerLine` to reassemble ordering.
   `orderSubtotal`, `orderDiscount`, `orderTotal` and `exciseTax` are **order-level** totals repeated
   on every row — sum the line fields, never the order fields.
3. **Pull invoices for the same window.** `InvoiceController_getInvoices` —
   `GET /v2/invoice?page=0&limit=500`, filterable by `orderNumber`, `irn`, `dispensary`,
   `organizationName`, `organizationId`, `startDate`/`endDate`, `startDueDate`/`endDueDate`.
   Join on `orderNumber` / `posoNumber`.
4. **Read the money fields precisely.** `subtotal`, `subtotalCollected`, `subtotalDue`, `due`,
   `total`, `tax`, `exciseTaxCollected`, `surcharge`, `discount`, and `overdue`. Credit memos come
   through as `creditMemo`, `creditMemoType`, `creditMemoNote`, `creditMemoCreatedAt`,
   `creditMemoCreatedBy`. **All numbers are plain JSON numbers** — Nabis documents that float,
   double and numeric are indistinguishable, so money is not in minor units and you must handle the
   floating-point yourself.
5. **Pull the retailer for credit context.** `RetailerController_getRetailer` —
   `GET /v2/retailer/{id}`, or list with `RetailerController_getRetailers` —
   `GET /v2/retailer?page=0&limit=500&isOnMarketplace=true`. The credit picture is
   `creditRating`, `previousCreditRating`, `previousCreditRatingDate`; compliance identity is
   `licenses`, `taxIdentificationNumber`, `sellerPermitLink`, `w9Link`; the corporate grouping is
   `parentOrganizationId` / `parentOrganizationName` (there is no `/organization` route — this is
   the only way to roll locations up to a parent).
6. **Cross-check payment risk on the order.** `paymentStatus`, `paymentTerms`,
   `daysTillPaymentDue`, `invoiceDueDate`, `paidAt`, `remittedAt`, `mustPayPriorBalance`,
   `lastNonPaymentDate`, `lastNonPaymentReason`, `retailerCreditRating` (NY orders).
   `mustPayPriorBalance` is the gate that blocks a new delivery on an outstanding balance.
7. **Before you flag a delivery gap, check the calendar.** `NabisDaysOffController_getNabisDaysOff` —
   `GET /v2/nabis-days-off` (no parameters). Nabis does not deliver at weekends or on the dates this
   route returns, so **zero orders on those dates is correct, not a data problem.**
8. **New York is a parallel tree.** `NYOrderController_getNyOrders` — `GET /v2/ny/order`,
   `NYInvoiceController_getNyInvoices` — `GET /v2/ny/invoice`,
   `NYRetailerController_getNyRetailers` — `GET /v2/ny/retailer`. NY orders add `pocContacts`,
   `orderTaxAmount`, `invoiceDueDate` and `retailerCreditRating` over the California shape. Run
   the two reconciliations separately.

## Rules that apply to every step

- `page` and `limit` are required on every paginated route; pagination is 0-indexed; drive the loop
  off `nextPage` until it is `null`; you never get more than 500 records per call.
- Dates in are `YYYY-MM-DD` only. Timestamps out are ISO 8601 in UTC.
- 15 requests per minute per key. A 90-day, multi-entity reconciliation is a multi-minute job —
  pace it rather than parallelising into a `429`.
- Errors are `{"statusCode", "message"}` — not RFC 9457, and with no stable error code. See
  `errors/nabis-problem-types.yml`.
