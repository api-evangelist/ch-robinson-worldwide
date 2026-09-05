---
name: Find and book a C.H. Robinson load as a carrier
description: Search C.H. Robinson's available shipment store, offer on a load or book it outright, then keep the load updated with milestones and documents.
api: openapi/ch-robinson-worldwide-rest-apis-openapi.yml
operations:
  - Generate Token
  - Available Shipment Search
  - Load Offer
  - Load Book
  - Milestone Updates
  - Carrier Document Upload
  - Payment Status Retrieval
---

# Carrier: find, book, update, get paid

The carrier side of the contract. C.H. Robinson offers this connectivity to carriers free of
charge; you still need issued credentials — start at https://developer.chrobinson.com/carrier.

## 1. Token

`Generate Token` — `POST /v1/oauth/token`, `grant_type: client_credentials`,
`audience: https://inavisphere.chrobinson.com`. JWT, 24-hour lifetime, cache it.

## 2. Find work

`Available Shipment Search` — `POST /v2/shipments/available/searches`. Returns `200`.

A search POST, not a GET — the filter goes in the body.

## 3. Offer or book

Two different commitments, pick deliberately:

- `Load Offer` — `POST /v1/shipments/{loadNumber}/offers`. Returns `202` (accepted, not
  decided) and may return `422`. The decision comes back asynchronously on your
  **Offer Response Callback** endpoint.
- `Load Book` — `POST /v1/shipments/books`. Returns `202` / `422`. Use this when the load
  already carries costs and you are taking it.

> **`Load Book` has no reversal in the contract.** There is no cancel or unbook operation.
> Once accepted, backing out is a phone call to C.H. Robinson or carrier support. Of every
> write in this API this is the one an agent must not fire speculatively — and because there
> is no idempotency key, a blind retry after a timeout is a second booking, not the same one.

On booking, C.H. Robinson pushes load details to your **Shipment Details Callback** endpoint so
you can build the load in your own TMS without polling.

## 4. Keep the load updated

`Milestone Updates` — `POST /v1/shipments/milestones`. Returns `201`. Append-only: there is no
operation to retract a milestone you sent.

## 5. Documents and money

- `Carrier Document Upload` — `POST /v1/documents/{loadNumber}`. Returns `201`. Posting
  paperwork here is what speeds up invoicing.
- `Payment Status Retrieval` — `GET /v1/financials/payments`. Check settlement status from
  inside your own TMS.

## Callbacks you host

Register these during onboarding; C.H. Robinson calls them with a bearer JWT (`bearerAuth`):

| Callback | Path in contract | What arrives |
|---|---|---|
| Offer Response Callback | `POST /offerResponse/callback/here` | outcome of a `Load Offer` |
| Shipment Details Callback | `POST /shipmentDetails/callback/here` | load details after booking |
| Load Documents Callback | `POST /events/LoadDocuments/here` | documents available for a load |
| Events Callback | `POST /REPLACE_ME_WITH_YOUR_CALLBACK_URL` | order and shipment events |

No retry policy and no payload signature scheme are published. Verify the bearer token; answer
`2XX` quickly and process asynchronously.

## EDI is a peer channel, not a legacy one

If you already run X12, C.H. Robinson publishes its own implementation guidelines for 204 load
tender, 990 tender response, 214 shipment status and 210 invoice at
`https://api.navisphere.com/api/B2B/Portal/v1/documents`, with AS2 and hosted SFTP configured
through https://developer.chrobinson.com/file-transfer. That path needs no REST integration.
