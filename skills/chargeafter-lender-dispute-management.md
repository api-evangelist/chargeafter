---
name: ChargeAfter lender dispute management
description: >-
  Run the lender-side chargeback and dispute lifecycle on ChargeAfter — open a dispute idempotently, assign
  it between merchant and lender, attach evidence, resolve or reopen it, and reverse the chargeback
  transaction.
api: openapi/chargeafter-disputes-management-lenders-openapi.yml
operations:
  - createADispute
  - getDisputeDetails
  - assignTheDisputeToMerchantOrLender
  - addNotesAndOrUploadSupportingDocumentsForADispute
  - resolveADispute
  - reopenAResolvedDispute
  - reverseTheChargebackTransaction
generated: '2026-08-09'
method: generated
---

# ChargeAfter lender dispute management

This is a **lender-facing** API. It is not linked from the ChargeAfter documentation navigation — it was
found through the published `/.well-known/api-catalog` linkset. Everything below is grounded in
`openapi/chargeafter-disputes-management-lenders-openapi.yml`, served from
`https://api.chargeafter.com` (sandbox `https://api-sandbox.ca-dev.co`), authenticated with
`Authorization: Bearer <PRIVATE_API_KEY>`.

Note the paths are unversioned — `/api/disputes/lenders`, with no `/v2` or `/v3` segment — so they carry no
version guarantee from the ChargeAfter change-guidelines policy.

## The one idempotent operation in the whole API

**`PUT /api/disputes/lenders`** (`createADispute`) is the only operation ChargeAfter documents as
idempotent, anywhere across its 46 published operations. The key is `lenderDisputeId`, a client-supplied
business key that must be unique within a single lender:

> "This call is idempotent. If there is an existing dispute with the same `lenderDisputeId` sent in a
> request, its ChargeAfter dispute `id` will be returned."

So:

- **Always set `lenderDisputeId`** from your own system, deterministically. Then a retry after a timeout is
  safe and returns the same ChargeAfter dispute `id`.
- On success the dispute is **automatically assigned to the Merchant**. Do not issue a separate assign call
  to achieve that; you would only be re-assigning what is already assigned.
- This guarantee does **not** extend to any other operation here. Resolve, reopen, reverse, update and
  assign have no idempotency key.

## Steps

1. **Open the dispute.** `PUT /api/disputes/lenders` (`createADispute`) with your `lenderDisputeId`.
   Returns the ChargeAfter dispute `id`. It lands assigned to the merchant.

2. **Read state before every subsequent write.** `GET /api/disputes/lenders/{id}` (`getDisputeDetails`).
   The lifecycle operations are not idempotent, so the current state is what tells you whether a write is
   still needed.

3. **Attach evidence.** `POST /api/disputes/lenders/{id}/update`
   (`addNotesAndOrUploadSupportingDocumentsForADispute`) — adds notes and/or uploads supporting documents.
   Repeated calls add more, they do not replace; do not re-send after a timeout without reading the dispute
   first.

4. **Move it between parties.** `PUT /api/disputes/lenders/{id}/assign`
   (`assignTheDisputeToMerchantOrLender`) — assigns the dispute to the Merchant or the Lender. This is the
   handoff that drives who is expected to act next.

5. **Close it.** `POST /api/disputes/lenders/{id}/resolve` (`resolveADispute`).

6. **Reopen if new evidence arrives.** `POST /api/disputes/lenders/{id}/reopen`
   (`reopenAResolvedDispute`) — only valid against a resolved dispute.

7. **Reverse the chargeback.** `PUT /api/disputes/lenders/{id}/reverse`
   (`reverseTheChargebackTransaction`). This moves money back. It has no idempotency key. Read the dispute
   with `getDisputeDetails` immediately before and immediately after, and never retry it blind.

## Rules an agent must follow

- **`reverse` and `resolve` are terminal, money-affecting, and not idempotent.** An agent should propose
  them and let a human confirm. Do not fire them on a retry loop.
- **Dispute documents are consumer evidence.** Uploaded supporting documents routinely contain PII and
  transaction detail. Do not copy them into logs, summaries, or third-party stores.
- **Errors** use the standard ChargeAfter envelope — `{ requestId, errors: [ { code, description } ] }` on
  400, `{ requestId, message, errors: [...] }` on 422. Codes `10000` invalid request payload, `10001` field
  value is required, `10002` field value is invalid, `1002` resource not found are all declared on this API.
  See `errors/chargeafter-error-codes.yml`.
- **This definition is not in the docs navigation.** Confirm with your ChargeAfter contact that you are
  entitled to this surface before integrating against it.
