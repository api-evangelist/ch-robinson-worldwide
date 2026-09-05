---
name: Track C.H. Robinson shipments and reconcile invoices
description: Consume C.H. Robinson order and shipment events (webhook or polling), pull visibility milestones, and reconcile the invoice and its PDF.
api: openapi/ch-robinson-worldwide-rest-apis-openapi.yml
operations:
  - Generate Token
  - Event Retrieval
  - Visibility GET Shipments
  - Visibility GET Milestone
  - Invoice Summary Retrieval
  - Invoice Retrieval
  - Invoice Document Retrieval
  - Load Document ID Retrieval
  - Load Document Retrieval
---

# Track, then reconcile

## 1. Token

`Generate Token` — `POST /v1/oauth/token`. 24-hour JWT, cached.

## 2. Choose your event channel

C.H. Robinson states the tradeoff itself: the callback is "preferred and most real-time",
polling is "easiest to implement and less real-time".

- **Push**: host the **Events Callback** endpoint. C.H. Robinson POSTs order and shipment
  events to it with a bearer JWT. The URL is registered during onboarding — the contract path
  is literally `/REPLACE_ME_WITH_YOUR_CALLBACK_URL`.
- **Pull**: `Event Retrieval` — `GET /v2/events`. Returns `200`. There is no pagination
  contract, so window your reads by the filter parameters rather than by page.

## 3. Visibility

- `Visibility GET Shipments` — `GET /v2/visibility/shipments`
- `Visibility GET Milestone` — `GET /v2/visibility/milestones`

To put a shipment, order or invoice under visibility in the first place use
`Visibility Shipment Create` (`POST /v2/visibility/shipments`),
`Visibility Order Create` (`POST /v2/visibility/orders`) or
`Visibility Invoice Create` (`POST /v2/visibility/invoices`). Only the shipment has a delete:
`Visibility Shipment Delete` — `DELETE /v2/visibility/shipments/{shipmentNumber}`.

## 4. Reconcile the money

1. `Invoice Summary Retrieval` — `GET /v1/financials/invoiceSummaries`. All active invoices in
   one call.
2. `Invoice Retrieval` — `GET /v1/financials/invoiceSummaries/{invoiceNumber}` for the detail.
3. `Invoice Document Retrieval` — `GET /v1/documents/invoices/{invoiceNumber}` for the PDF.

## 5. Supporting documents

`Load Document ID Retrieval` — `GET /v1/documents/retrieveDocumentIds`, then
`Load Document Retrieval` — `GET /v1/documents/{documentId}` for signed BOLs, PODs and lumper
receipts. Blank BOLs are generated with `GENERATED BOL RETRIEVAL - Order Level`
(`GET /v1/documents/generateBols`) or `... - Load Level` (`GET /v1/documents/generateLoadBols`).

## Reading errors

Everything here is a read, so the failure modes are narrow: `401` means your 24-hour token
expired, `403` usually means the `customerCode` is not yours (`{"statusCode":403,
"error":"Forbidden","message":"Invalid customerCode"}`), `404` means the reference does not
exist. `500` is undocumented — capture the `correlation-id` response header and send it to
ebizhelpdesk@chrobinson.com.
