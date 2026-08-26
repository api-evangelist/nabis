---
name: nabis-trace-order-manifest
description: Resolve a Nabis order to its regulatory transfer manifest and the QR codes of every scanned case and item, using the Universal Cannabis API (UCAPI) labeling routes — including the caveat that these documented routes did not answer on the public host when last probed.
api: Nabis Platform API v2
base_url: https://platform-api.nabis.pro
generated: '2026-08-26'
method: generated
source: openapi/nabis-platform-api-v2-openapi.yml + https://github.com/Cannabis-Labeling-API/universal-cannabis-api
operations:
  - UniversalCannabisApiController_wellKnown
  - UniversalCannabisApiController_getOrder
  - UniversalCannabisApiController_getManifest
  - OrderController_getOrder
---

# Trace a Nabis order to its cannabis transfer manifest

Nabis implements the **Universal Cannabis API (UCAPI)** — a multi-vendor standard for cannabis
labeling provider integrations — under the `/ucapi/` namespace, separate from the `/v2/` business
API. The point of the standard is that a retailer can scan **one** barcode on a box and then resolve
every case and item inside it through whichever vendor's API applies, without a bespoke connector
per distributor.

## Read this first — verification status

When probed anonymously on **2026-08-26**, `GET https://platform-api.nabis.pro/ucapi/.well-known/cannabis-api.json`
returned **404** with `{"statusCode":404,"message":"Cannot GET /ucapi/.well-known/cannabis-api.json"}`.
That is a *router-level* 404, not the `401` that auth-gated `/v2/` routes on the same host return —
so the documented UCAPI surface may not be deployed on this host, may sit behind a different
hostname, or may have been withdrawn while its reference pages remain published.
**Confirm reachability with Nabis (help@nabis.com) before you build on these three operations.**
Everything below is grounded in the published contract, not in a successful live call.

## Steps

1. **Discover the endpoint.** `UniversalCannabisApiController_wellKnown` —
   `GET /ucapi/.well-known/cannabis-api.json`. This is the standard's own discovery document. It
   returns three required fields:
   - `endpoint` — the root endpoint for every other route in the document
   - `path-components` — the path components for each of the endpoints
   - `vendor` — always the string `Nabis`, which is how a client matches a stored API key to the
     right vendor when it is talking to several distributors.

   Use the returned `endpoint`; do not hard-code the base.
2. **Resolve the order.** `UniversalCannabisApiController_getOrder` — `GET /ucapi/order/{orderId}`
   where `orderId` is the order UUID (the same `id` returned by `OrderController_getOrder` on
   `GET /v2/order/{id}`). The response is the regulatory view of the order and every field is
   required:
   - `orderId` (uuid), `deliveryDate`, `fulfillmentDate`, `licenseNumber`
   - **`metrcManifestId`** (number) — the state track-and-trace transfer manifest identifier. This
     is the join key between the Nabis order and the Metrc record; note it is a **number**, not a
     UUID, because it originates in the state system, not in Nabis.
   - `lineItems` — `quantity`, `pricePerUnit`, `standardPricePerUnit` (the un-discounted wholesale
     unit price from the Inventory tab at order-creation time), `isExciseTaxable`, `discount`.
3. **Pull the manifest.** `UniversalCannabisApiController_getManifest` —
   `GET /ucapi/manifest/{orderId}`. Returns a `regulatorEvent` object:
   - `id` — the regulator ID
   - `kind` — always the string `manifest`
   - `contents` — an array of QR codes representing **all scanned cases or items**
4. **Expand the contents.** Per the standard, each QR code in `contents` is looked up against the
   case-information endpoint of the vendor that issued it to obtain the `each` IDs, and from those
   the individual scanned-out packages. Nabis publishes only the well-known, order and manifest
   routes; the `product`, `batch`, `case`, `each` and `regulator` routes the standard defines are
   **not** in the Nabis contract, so a full expansion needs the issuing vendor's own UCAPI
   implementation.

## Related artifacts

- Nabis also publishes its own OpenAPI 3.0.0 domain model for this problem space — "Universal QR
  Code API", contact `engineering@nabis.com` — in its public GitLab group. It is explicitly a
  "Sample API" with no `servers` block, i.e. a design artifact:
  `openapi/nabis-universal-qr-code-openapi.yaml` (source:
  https://gitlab.com/nabis-engineering/core/universal-qr-codes).
- Nabis's status page tracks Metrc as a monitored dependency component: https://status.nabis.com.
- Conformance evidence and the full standard mapping: `conformance/nabis-conformance.yml`.

## Failure modes

`401 Invalid API key` — send `x-nabis-access-token`. `404 Record not found` — the order UUID does
not resolve for your organization. `429 Too Many Requests` — 15/min ceiling. All three operations
are `GET`; retries are safe and nothing here can be undone because nothing here writes.
