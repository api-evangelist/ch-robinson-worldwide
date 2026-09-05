---
name: Quote to order with C.H. Robinson
description: Get a freight rate from C.H. Robinson and turn the quote into a booked order, using the quote ID so the price you were shown is the price you tender.
api: openapi/ch-robinson-worldwide-rest-apis-openapi.yml
operations:
  - Generate Token
  - Rating Request
  - Order Create with Quote ID
  - Order Delete
---

# Quote to order

Use this for a shipper/customer flow: price a shipment, then tender it.

## 1. Get a token

`Generate Token` — `POST /v1/oauth/token`.

Body: `client_id`, `client_secret`, `audience: https://inavisphere.chrobinson.com`,
`grant_type: client_credentials`. Send `Content-Type: application/json`.

The token is a JWT valid for **24 hours**. Cache it. C.H. Robinson explicitly warns that calling
this endpoint more than once per 24 hours may get you rate limited. Present it as
`Authorization: Bearer <jwt>` on every other call.

Base URL: `https://sandbox-api.navisphere.com` while testing, `https://api.navisphere.com` in
production. Nothing in the credential tells you which one it belongs to — track that yourself.

## 2. Ask for a rate

`Rating Request` — `POST /v1/quotes`. Returns `201` with `quoteSummaries[]`.

Each summary carries `quoteId`, `totalCharge`, `totalFreightCharge`, `totalAccessorialCharge`,
`transit`, a `rates[]` breakdown (rate code `400` line haul, `405` fuel surcharge) and, on
carrier-select quotes, `carrier.scac`.

The more detail you send, the tighter the quote. If you do not know the LTL freight class, call
`Estimating Freight Class` (`POST /v1/quotes/freight-class-estimate`) first — it applies NMFTA
density-based classification.

## 3. Tender the quote

`Order Create with Quote ID` — `POST /v1/orders/quotes`. Returns `201`.

Pass the `quoteId` you were given. This is the path that preserves the quoted price.

Two rules from C.H. Robinson's own reference:
- For shipments with more than two stops, this endpoint accepts **only** the quote ID and no
  supplemental shipment data.
- Orders missing any field marked required are created **incomplete** and need a human at
  C.H. Robinson to finish them.

If you need to send shipment data alongside the rate, use `Rating Request` then
`Order Create` (`POST /v1/orders`) instead — see the Rating Workflow Options Guide.

## 4. If you need to back out

`Order Delete` — `DELETE /v1/orders/{orderNumber}`.

**There is no published cancellation window.** Do not assume one. Treat order creation as
reversible in principle and unbounded in practice, and confirm with the C.H. Robinson
representative for anything time-sensitive.

## Rules that apply to every step

- **No idempotency key exists.** If `POST /v1/orders/quotes` times out, do not blind-retry —
  you can create a duplicate freight order. Read back with the customer reference number first.
- **No pagination.** Narrow collection reads with filters, not pages.
- **Errors are not RFC 9457.** `400` returns an *array* of `{message, path[], type, context}`
  validation objects; `401`/`403`/`404` return `{statusCode, error, message}`; `500` has no
  documented body and the description tells you to contact your C.H. Robinson representative.
- **Rate limits arrive as headers**, not in the docs: `RateLimit-Limit`, `RateLimit-Remaining`,
  `RateLimit-Reset`. Back off on `429`.
- **Check health first** for anything expensive: `GET https://api.navisphere.com/api/B2B/Portal/v1/statuses`
  is unauthenticated and returns `{name, healthy}` for Rating, Orders, Authentication and the rest.
