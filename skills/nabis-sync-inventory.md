---
name: nabis-sync-inventory
description: Pull a complete, correctly paginated snapshot of a brand's inventory from the Nabis Platform API v2 (California or New York), and interpret the quantity fields correctly given that they are a cached sync, not a live warehouse count.
api: Nabis Platform API v2
base_url: https://platform-api.nabis.pro
generated: '2026-08-26'
method: generated
source: openapi/nabis-platform-api-v2-openapi.yml + https://developers.nabis.com/v2/docs/overview/inventory-quantities
operations:
  - InventoryController_getInventories
  - InventoryController_getInventory
  - NYInventoryController_getNyInventory
  - NYInventoryController_getNyInventoryByItemCode
  - NYInventoryController_getNyInventoryAggregate
  - InventoryHistoryController_getInventoryHistoryForSnapshotDate
  - InventoryHistoryController_getInventoryHistoryForSkuBatch
---

# Sync Nabis inventory

## Before you start

- Authenticate with the header `x-nabis-access-token: <key>`. There is no OAuth, no bearer token
  and no sandbox — the key reads live production data for one organization.
- **You get 15 requests per minute.** A full inventory sync at `limit=500` costs one request per
  500 rows. Budget the whole job against that ceiling before you start; there is no `Retry-After`
  header to back off on, only a `429` with `{"statusCode":429,"message":"Too Many Requests"}`.
- California and New York are **different route trees**, not a parameter. Pick one.

## Steps

1. **List page 0.** `InventoryController_getInventories` — `GET /v2/inventory?page=0&limit=500`.
   Both `page` and `limit` are REQUIRED; omitting either returns `400 Bad request exception`.
   Pagination is **0-indexed**.
2. **Read the envelope, not the array length.** The response is
   `{data, page, totalCount, totalNumPages, nextPage, prevPage}`. Drive the loop off `nextPage`
   and stop when `nextPage` is `null`. Do **not** loop on `totalNumPages`: it is 0-indexed
   (42 real pages reports `41`) *except* when there is exactly one page, where it reports `1`.
   `nextPage` has no such edge case.
3. **Cap at 500.** `limit` accepts up to 1000 but no call ever returns more than 500 records, and a
   `limit` above 1000 returns `400`. Asking for 750 silently gives you 500.
4. **For New York**, use `NYInventoryController_getNyInventory` — `GET /v2/ny/inventory?page=0&limit=500`.
   The NY row carries five fields California does not: `skuId`, `isMedical`,
   `availableOnMarketplace`, `deletedAt`, `licenseNickname`. Map them separately; the schemas are
   duplicated, not shared.
5. **For one product**, use `InventoryController_getInventory` — `GET /v2/inventory/{skuCode}`
   (e.g. `FG1500-WAVERIDER`), or `NYInventoryController_getNyInventoryByItemCode` —
   `GET /v2/ny/inventory/{itemCode}`. These take no pagination parameters.
6. **For a point in time**, use `InventoryHistoryController_getInventoryHistoryForSnapshotDate` —
   `GET /v2/inventory-history/snapshot-date/{snapshotDate}?page=0&limit=500`. The date must be
   `YYYY-MM-DD` and nothing else; any other format returns
   `400 Invalid ISO8601 date provided for snapshot date. Format: 'YYYY-MM-DD'. Example: '2022-04-20'`.
   To trace one lot over time, use
   `InventoryHistoryController_getInventoryHistoryForSkuBatch` — `GET /v2/inventory-history/sku-batch/{skuBatchId}`.

## Interpreting the quantities — read this before you act on them

`warehouseCount` is **not real time**. Nabis documents it as a cached sync that may lag
**30 minutes or more** behind the warehouse system.

- `warehouseCount.updatedAt` records **when the sync last detected a difference**, not when the sync
  last ran. An old `updatedAt` means the number has been *stable*, not that it is *stale*. Never use
  `updatedAt` alone as a freshness signal.
- Nabis's own guidance: treat quantities as a **soft signal**, build in a buffer, and do not use them
  as a hard availability guarantee for order logic.
- Non-sellable states are distinct and are not all visible here: **archived** items (removed from
  sellable state by a brand or retailer) are excluded from this API's results entirely;
  **quarantined** is an operational state Nabis controls, typically pending lab results; **expired**
  (COA past its expiration date) is a state, not a field, and is not exposed.

## Failure modes

| Status | Body | What to do |
|---|---|---|
| 400 | `Bad request exception` | You omitted `page` or `limit`, or sent `limit > 1000`. |
| 400 | `Invalid ISO8601 date provided…` | Reformat the date as `YYYY-MM-DD`. |
| 401 | `Invalid API key` / `Unauthorized` | Re-copy the key from the Nabis app, Team → API. |
| 429 | `Too Many Requests` | You exceeded 15/min. Wait out the minute; there is no `Retry-After`. |
| 500 | `Unable to fetch inventory data…` | The backing warehouse service was unreachable. Safe to retry — this is a GET. |
| 408 | `Your request is taking over 60000ms…` | Narrow the query. The server budget is 60 seconds. |
| 503 | `The API is currently under maintenance…` | Check https://status.nabis.com. |

Every operation in this skill is a `GET`. Retries are always safe and there is nothing to undo.
