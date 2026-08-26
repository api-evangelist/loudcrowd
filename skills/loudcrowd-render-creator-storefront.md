---
name: loudcrowd-render-creator-storefront
description: >-
  Build a custom creator storefront by reading ambassador, collection, feed, media and product data
  from the Creator Storefronts API instead of embedding LoudCrowd's hosted web components.
api: LoudCrowd Creator Storefronts API
generated: '2026-08-25'
method: generated
source: >-
  openapi/loudcrowd-creator-storefronts-openapi.yml,
  https://docs.loudcrowd.com/docs/install-the-loudcrowd-storefront-page-in-your-custom-website
operations:
  - GET /StorefrontAmbassador
  - GET /StorefrontCollections
  - GET /StorefrontFeedItems
  - GET /StorefrontMediaDetails
  - GET /StorefrontProductDetails
---

# Render a creator storefront from the API

Use this only if you are **not** using the hosted web components. LoudCrowd is explicit that this API
exists in place of them and that "the user of this api is responsible for rendering the data in a way
that is consistent with the creator storefronts." If you just want a storefront, install the SDK
instead — see `components/loudcrowd-components.yml`.

> **Check the base URL before you build.** The published `servers[]` value is
> `https://store-api.loudcrowd.com/api`, and that host did **not resolve in DNS** when probed on
> 2026-08-25. Confirm the current base with your LoudCrowd representative before writing a client
> against it.

Authentication is an HTTP bearer token: `Authorization: Bearer <token>`.

## Identifiers

Every operation is keyed on `lcAmbassadorId` — an opaque base64-shaped string, max 32 characters. It
is the same value the Brand API returns as `ambassadorLcId` on a `brandOrder`, which is how you get
from an attributed order back to the creator who drove it.

Shop context is supplied as **either** `shopId` **or** `shopifyId`. Send one; neither is required if
the other is present.

## Steps

1. **`GET /StorefrontAmbassador?lcAmbassadorId=...`** — ambassador identity, including the
   `ecommAmbassador.affiliateCode`. Render the storefront header from this.

2. **`GET /StorefrontCollections?lcAmbassadorId=...`** — the collections this ambassador curates.
   Optionally narrow with `sharedCollectionId` when you are rendering a deep link to one collection.

3. **`GET /StorefrontFeedItems?lcAmbassadorId=...&storefrontCollectionId=...`** — the collection body.
   This is the heaviest operation, with fifteen query parameters.

   - **Two independent cursors.** The feed interleaves shoppable media and picked products from two
     sources, so `mediaCursor` and `productCursor` advance separately. Carry **both** forward on
     every subsequent page; advancing only one silently truncates the other stream.
   - Filter with `includeFeedItemTypes`, `mediaTypes`, `minMediaPostedAt`, `includeTaggedMedia`,
     `includeUntaggedMedia`, `includeHiddenMedia`; order with `mediaSort`.
   - Deep links into a single item use `sharedMediaId` / `sharedProductId`.

4. **`GET /StorefrontMediaDetails?lcAmbassadorId=...&mediaId=...`** — everything needed to render one
   shoppable post, including its `taggedProductItem` entries.

5. **`GET /StorefrontProductDetails?lcAmbassadorId=...&productVariantId=...`** — variant-level detail
   for a picked product, including `productPrices`, `priceCurrencyCode` and `selectedOptionItem`
   values for the option pickers.

## Attribution is still your job

These are read operations. They do not attribute anything. Purchases are attributed either by the
affiliate link structure LoudCrowd generates
(https://docs.loudcrowd.com/docs/creator-storefront-affiliate-link-structue) or by posting order
events server-side — see `loudcrowd-submit-order-events`. A storefront rendered from this API with no
attribution path wired up will show creator content and credit nobody.

## Errors

`400` and `404` only; envelope is `{code, message}`. A `404` means the ambassador, collection, media
or variant does not exist for that shop — check that you sent the right `shopId`/`shopifyId` pairing
before assuming the id is wrong.
