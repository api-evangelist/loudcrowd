---
name: loudcrowd-process-returns
description: >-
  Report returns and refunds to LoudCrowd so creator commissions are adjusted correctly, using
  stable cumulative refund line item ids instead of incremental refund events.
api: LoudCrowd Attribution Events API
generated: '2026-08-25'
method: generated
source: openapi/loudcrowd-openapi.yml, https://docs.loudcrowd.com/reference/submit_return_event
operations:
  - submit_order_event
---

# Report returns and refunds to LoudCrowd

Returns are not a separate endpoint or topic. They ride on the order event.

- Endpoint: `POST https://api.loudcrowd.com/event/ecomm`
- Topic: `X-LC-TOPIC: ORDER_UPDATE`
- Auth and payload basics: see `loudcrowd-submit-order-events`.

## The one rule that matters

**Assign one stable `refund_line_item_id` per original `line_item_id`, and always send the current
CUMULATIVE totals for it.**

- Unique within the ecommerce integration, and stable for that original line item forever.
- Reuse it on every later update, sending the current cumulative `amount`, `tax` and `quantity`
  refunded — not the increment.
- Use a *different* refund id only for a *different* original line item. Issuing a second refund id
  for the same original line item double-counts the return and over-claws the creator's commission.

Because the value is cumulative rather than incremental, a mis-stated refund is corrected by
restating it — including restating it back to zero. That is the only undo available.

## Steps

1. **Rebuild the full order payload.** Include every original `line_items` entry plus the new
   `refund_line_items`.

2. **Leave the order totals gross.** Keep `amount`, `tax` and `shipping` at the order's pre-refund
   values. LoudCrowd subtracts the refund amounts separately. Reducing them yourself subtracts the
   refund twice.

3. **Advance `platform_updated_at`** to the current timestamp.

4. **Send with `X-LC-TOPIC: ORDER_UPDATE`**, signed over the exact raw bytes as in
   `loudcrowd-submit-order-events`.

5. **Confirm HTTP `200` with the plain-text body `success`.**

## Refund line item shape

```json
{
  "refund_line_item_id": "REFUND_001",
  "line_item_id": "1",
  "platform_refunded_at": "2021-01-03T10:00:00Z",
  "sku": "SKU1",
  "variant_sku": "VARIANT1",
  "amount": "100.00",
  "tax": "6.00",
  "quantity": "2"
}
```

## How commission is adjusted

- **Full refund** — refund amount equals the line item amount: the whole line item is disqualified
  for commission.
- **Partial refund** — refund amount is less than the line item amount: commission is adjusted
  proportionally.
- **Multiple refunds against one line item** — update that line item's existing refund record with
  cumulative values. Do not create a second record.

## Verifying the result

The `200 success` response confirms acceptance only. To see the adjusted figures, poll
`list_program_orders` and read `refundAmount` / `refundTax` on the `brandOrder`, which are reported
separately from `amount` and `tax`.
