---
name: loudcrowd-pull-attributed-orders
description: >-
  Pull every order LoudCrowd has attributed to members of a program, page through the cursor
  correctly, and reconcile commission and refund values without double-counting.
api: LoudCrowd Brand API
generated: '2026-08-25'
method: generated
source: openapi/loudcrowd-openapi.yml, https://docs.loudcrowd.com/reference/list_program_orders
operations:
  - list_program_orders
---

# Pull attributed orders from a LoudCrowd program

The Brand API has exactly one operation. Everything below is that one operation used correctly.

## Before you start

- Base URL is `https://api.loudcrowd.com`.
- Create a token in the LoudCrowd app under **Settings → API Keys** with the **Read orders** and
  **Read programs** scopes. The token is shown once; store it in a secret manager.
- `account_id` and `program_id` are shown in the app under **API identifiers**. They are opaque
  strings — do not construct them.

## Steps

1. **Call `list_program_orders`.**

   `GET /api/v1/brand/{account_id}/programs/{program_id}/orders`
   with header `X-LC-Account-Key: <token>`.

   Optional query parameters: `limit` (1–100, default 50) and `cursor`.

2. **Page until `nextCursor` is null.** The response body is
   `{ "data": [ brandOrder, ... ], "nextCursor": "..." | null }`. A `Link` header with `rel="next"`
   is also returned and is absent on the final page. Either signal works; `nextCursor === null` is
   the definitive terminator. Pass the returned cursor back as `cursor` on the next call and change
   nothing else.

3. **Understand the ordering before you build an incremental sync.** Results come back in the order
   *LoudCrowd received them*, most recent first — that is receipt order, not `orderedAt` order. A
   backfilled or late-arriving order can appear ahead of a newer one. Do not treat position in the
   list as a timestamp, and do not stop paging early because you saw an order you already have.

4. **Read each `brandOrder` correctly.** Fields: `orderId`, `ambassadorLcId`, `attributionTypes[]`,
   `commissionAmount`, `amount`, `tax`, `shipping`, `refundAmount`, `refundTax`, `currency`,
   `cancelledAt`, `orderedAt`, `lastUpdatedAt`.

   - `amount`, `tax` and `shipping` are **gross, pre-refund** values. `refundAmount` and `refundTax`
     are separate. Net revenue is `amount - refundAmount`; never assume `amount` was already reduced.
   - Money fields are decimal **strings** (`"100.00"`). Parse them as decimals, not floats.
   - `currency` is ISO 4217 — do not aggregate across currencies.
   - `cancelledAt` non-null means the order was cancelled; exclude it from commission totals.
   - `ambassadorLcId` is the join key to every Creator Storefronts operation, where the same value is
     called `lcAmbassadorId`.

5. **Handle failures per `errors/loudcrowd-problem-types.yml`.** Errors are `{code, message}` JSON,
   not RFC 9457. Retry `429` and `5xx` with capped exponential backoff and jitter, honouring
   `Retry-After` when present. Do **not** retry `400` or `401` without changing the request — a `401`
   here means the token or its scopes are wrong, and retrying will not fix it.

## What this API cannot do

There is no push notification when attribution resolves or a commission is calculated. This
operation is a polling loop by design. Pick an interval, and re-poll a trailing window rather than
only the head of the list, because receipt order is not chronological order.
