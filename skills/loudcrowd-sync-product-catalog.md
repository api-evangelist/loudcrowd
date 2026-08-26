---
name: loudcrowd-sync-product-catalog
description: >-
  Keep the product catalog LoudCrowd shows to creators in sync, via the batch product-data API or
  the full-replacement CSV feed, without silently deactivating the rest of the catalog.
api: LoudCrowd Attribution Events API
generated: '2026-08-25'
method: generated
source: >-
  openapi/loudcrowd-openapi.yml,
  https://docs.loudcrowd.com/docs/send-product-data-to-loudcrowd
operations:
  - send_product_data
---

# Sync a product catalog into LoudCrowd

Product data is what creators pick from for their storefront collections and tag in their UGC. There
are two delivery paths and they have **different destructive semantics**. Choose deliberately.

## Path A — the batch API (incremental, safe)

`POST https://api.loudcrowd.com/product-data`, operationId `send_product_data`.

1. **Authenticate like an order event** — `X-LC-SHOP-ID`, `X-Signature` (HMAC-SHA256 of the exact raw
   body bytes), `X-LC-TOPIC`, `Content-Type: application/json`.

2. **Send `{ "products": [ ... ] }`** — between **1 and 1000** `productIngestRecord` items per call.
   That ceiling is hard; chunk larger catalogs.

3. **Populate each record.** Required: `platform_product_id`, `title`. Then `handle`, `brand`,
   `image_src`, `product_url`, `active`, `in_stock`, `price_currency_code` (ISO 4217), `prices`,
   `promotional_code`, `collections[]` (≤250), `tags[]` (≤250), `variants[]` (≤250).
   Use explicit `null` to clear an optional value.

4. **Read the response.** `200` returns `{ success, message, processedCount }`. Check
   `processedCount` against the batch size you sent.

5. **Handle `422` and `429`.** `422` means the batch was well-formed but unprocessable — fix the
   records, do not retry unchanged. `429` is the only rate-limit response declared anywhere in
   LoudCrowd's contract: back off and honour `Retry-After` when present.

This path is an **upsert on `platform_product_id`**. Records you omit are left alone.

## Path B — the CSV/SFTP feed (full replacement, destructive)

Uploaded to LoudCrowd's hosted SFTP, or pulled from one you host. Requires setup by a LoudCrowd
representative.

> **The feed file is treated as your entire catalog.** Anything present in a previous file and
> missing from a newer one is marked **inactive** and its data made unavailable. You cannot send an
> incremental file. A truncated export silently deactivates everything it omitted.

Columns: `productId`, `parentId`, `position`, `brand`, `title`, `active` (`Y`/`N`), `URL`,
`imageSrc`, `language` (ISO 639-1), `country` (ISO 3166-1 alpha-2), `listPrice`, `salePrice`,
`currencyCode`, `availability` (`in stock`/`out of stock`), `option1Name`/`option1Value` …

- Variants must appear on the rows **immediately after** their parent, with `position` starting at 1;
  parents and standalone products use `position` 0 and set `parentId` to their own `productId`.
- Multiple images: repeat the row, changing only `imageSrc`.
- Translations: repeat the variant row with different `language`/`country`.
- A daily file is recommended because products change often.

## Recovery

Neither path has a versioned rollback. Restore by re-upserting the affected records (Path A) or by
re-sending the **complete** catalog file (Path B). No time window is stated for either.
