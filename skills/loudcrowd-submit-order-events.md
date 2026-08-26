---
name: loudcrowd-submit-order-events
description: >-
  Send order create, update and cancel events from a custom or headless commerce stack to LoudCrowd
  for creator attribution and commission calculation, including correct HMAC-SHA256 request signing.
api: LoudCrowd Attribution Events API
generated: '2026-08-25'
method: generated
source: openapi/loudcrowd-openapi.yml, https://docs.loudcrowd.com/reference/submit_order_event
operations:
  - submit_order_event
---

# Submit order events to LoudCrowd

One endpoint, three topics. `POST https://api.loudcrowd.com/event/ecomm`.

## Prerequisites

- The LoudCrowd JS SDK is installed on the storefront (`https://pub.loudcrowd.com/embed.js`) and a
  shoppable program is generating affiliate links. Without it there is nothing to attribute to.
- An API token carrying the **Write orders** scope.
- Your **Shop ID** from the LoudCrowd **Integrations** page. This is *not* an Account ID and *not*
  an API token.

## Steps

1. **Build the complete order payload.** Send the whole order on every topic — this endpoint does
   not accept deltas.

   If the order-confirmation script is not installed, obtain the device tracking id in the browser
   with `window.loudcrowd.identifyDevice()` and include it as `lc_anon_id`.

   For gross-tax regions, send the tax-inclusive price as `amount` and the VAT as `tax`:
   `tax = round(amount - (amount / (1 + tax_rate)), 2)`.

2. **Serialize once, then sign those exact bytes.**

   ```python
   import hashlib, hmac
   from pathlib import Path

   api_token = "YOUR_API_TOKEN"
   body = Path("order.json").read_bytes()
   signature = hmac.new(api_token.encode("utf-8"), body, hashlib.sha256).hexdigest()
   ```

   Send `body` unchanged on the wire. **Re-serializing, pretty-printing or reordering keys after
   signing changes the bytes and the request will fail with `401`.** The digest is lowercase hex with
   no `sha256=` prefix.

3. **Set the headers.**

   | Header | Value |
   |---|---|
   | `X-LC-SHOP-ID` | your Shop ID |
   | `X-Signature` | the hex digest from step 2 |
   | `X-LC-TOPIC` | `ORDER_CREATE`, `ORDER_UPDATE` or `ORDER_CANCEL` |
   | `Content-Type` | `application/json` |

4. **Pick the topic.**

   - `ORDER_CREATE` — a new order.
   - `ORDER_UPDATE` — any restatement, including returns and refunds (see
     `loudcrowd-process-returns`).
   - `ORDER_CANCEL` — set `platform_cancelled_at` to the cancellation time **and** set
     `platform_updated_at` later than the preceding event. The topic header alone does not cancel
     the order; this is the documented failure mode.

5. **Verify the response body, not just the status.** Success is HTTP `200` with the plain-text body
   `success`. Anything else must be logged and investigated. Note what `success` means: the event was
   authenticated and accepted **for asynchronous processing**. It does not confirm persistence,
   attribution, or that a commission was calculated. To observe the outcome, poll the Brand API
   (`loudcrowd-pull-attributed-orders`).

6. **Retry safely.** There is no `Idempotency-Key` header. Replay-safety comes from reusing the same
   order id and refund ids — re-sending the same identifiers is an update, never a duplicate. On
   `429` or `5xx`, retry with capped exponential backoff and jitter, reusing those ids unchanged.

## Reversal

There is no delete or void. Correct an order by re-sending it with `ORDER_UPDATE`, or cancel it with
`ORDER_CANCEL`. LoudCrowd states no time window on either, so do not assume one exists — see
`conventions/loudcrowd-conventions.yml`.

## Operational advice from the provider

Send events as soon as orders are placed, and build a back-fill pipeline alongside the events API
rather than relying on live delivery alone.
