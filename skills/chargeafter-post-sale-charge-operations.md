---
name: ChargeAfter post-sale charge operations
description: >-
  Manage a ChargeAfter financing charge after it is created — settle on shipment, record delivery, refund a
  returned item, or void an authorization before settlement — without double-moving money on a retry.
api: openapi/chargeafter-charge-api-openapi.yml
operations:
  - createANewCharge
  - getDetailsOfACharge
  - updateACharge
  - settleACharge
  - refundACharge
  - voidACharge
  - updateDeliveryData
  - getChargeId
  - getTransactionByMerchantTransactionId
  - getTransactionById
generated: '2026-08-09'
method: generated
---

# ChargeAfter post-sale charge operations

Every operation here moves real consumer credit. Read the "Before any write" section before calling any of
them.

All operations are in `openapi/chargeafter-charge-api-openapi.yml` at
`https://api.chargeafter.com` (sandbox: `https://api-sandbox.ca-dev.co`), authenticated with
`Authorization: Bearer <PRIVATE_API_KEY>`.

## Before any write

ChargeAfter publishes **no idempotency key** for any charge operation — no `Idempotency-Key` header exists
anywhere in the API. A retried settle, refund, or void after a network timeout can move money twice, and
nothing in the contract will stop it.

So, before re-issuing any write:

1. Resolve current state. `GET /v2/post-sale/charges/{chargeid}` (`getDetailsOfACharge`) if you have the
   charge id; `GET /v2/post-sale/orders/{orderId}` (`getChargeId`) to resolve your own `merchantOrderId`
   into a ChargeAfter `chargeId`; or `GET /v2/post-sale/transactions/lookup`
   (`getTransactionByMerchantTransactionId`) when all you kept was your `merchantTransactionId`.
2. Only then decide whether the write actually needs to be repeated.

Always send your own `merchantOrderId` / `merchantTransactionId` on creation. They are the only handles that
let you answer "did that land?" later.

## Operations

### Create
- `POST /v2/post-sale/charges` (`createANewCharge`) — initiates a new charge transaction.
- `POST /v2/post-sale/consumers/{consumerid}/charges` — the consumer-scoped equivalent in
  `openapi/chargeafter-consumers-management-openapi.yml`.

### Read
- `GET /v2/post-sale/charges/{chargeid}` (`getDetailsOfACharge`) — current state of a charge.
- `GET /v2/post-sale/transactions/{id}` (`getTransactionById`) — a single transaction.
- `GET /v2/post-sale/transactions/lookup` (`getTransactionByMerchantTransactionId`) — resolve by your id.
- `GET /v2/post-sale/orders/{orderId}` (`getChargeId`) — resolve a merchant order to a charge id.

### Modify
- `PATCH /v2/post-sale/charges/{chargeid}` (`updateACharge`).
- `POST /v2/post-sale/charges/{chargeid}/items/delivery` (`updateDeliveryData`) — update delivery data for
  items on an existing charge. Many lenders will not fund until delivery is recorded, so this is part of
  getting paid, not an optional nicety.

### Move money
- `POST /v2/post-sale/charges/{chargeid}/settles` (`settleACharge`) — completes the payment by settling an
  authorized charge. Settle when you ship, not when you take the order.
- `POST /v2/post-sale/charges/{chargeid}/refunds` (`refundACharge`) — returns funds on a previously settled
  amount.
- `POST /v2/post-sale/charges/{chargeid}/voids` (`voidACharge`) — cancels an authorized charge **before**
  settlement, so it never reaches the consumer's statement. Void early rather than settle-then-refund
  whenever the transaction has not settled.
- `POST /v2/post-sale/charges/voids/async` — the asynchronous void surface. The result does not arrive in
  the response; confirm via `getDetailsOfACharge` before telling anyone the void succeeded.

Choose deliberately: **void** for a not-yet-settled authorization, **refund** for a settled amount. They are
not interchangeable and the consumer sees different things.

## Errors

- `2001` charge not found
- `2002` charge already has assigned confirmation — the charge exists; reconcile, do not retry
- `2004` confirmation not found
- `2006` operation is not permitted at this stage — you are trying to settle, refund or void a charge whose
  state does not allow it. Read the charge first rather than retrying.
- `1002` resource not found, `1005` field value is invalid, `1110` validation error

400 returns `{ requestId, errors: [ { code, description } ] }`; 422 returns
`{ requestId, message, errors: [ { field, validationState, error } ] }`. Log `requestId` every time — it is
the only trace handle ChargeAfter gives you, and only on failures. Full registry:
`errors/chargeafter-error-codes.yml`.

## Reconciliation

Money actually moving is visible in `GET /v2/post-sale/fundings/report`
(`openapi/chargeafter-funding-api-openapi.yml`), not in the charge record. See
`skills/chargeafter-funding-reconciliation.md`.
