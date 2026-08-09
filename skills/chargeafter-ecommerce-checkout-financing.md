---
name: ChargeAfter ecommerce checkout financing
description: >-
  Take a consumer from a cart to an approved point-of-sale financing charge using ChargeAfter's v3 session
  flow — create a session, read the lender offers, resolve the approved account, bind it to the cart with a
  confirmation, then create the charge.
api: openapi/chargeafter-checkout-session-openapi.yml
apis:
  - openapi/chargeafter-checkout-session-openapi.yml
  - openapi/chargeafter-checkout-session-accounts-openapi.yml
  - openapi/chargeafter-distribution-api-openapi.yml
  - openapi/chargeafter-checkout-application-status-openapi.yml
  - openapi/chargeafter-charge-api-openapi.yml
operations:
  - createANewSession
  - getLenderAccountsAndOffers
  - getAccountsWithOffersForConsumer
  - getStatusOfAnApplication
  - createConfirmationWithApplicationAndCartDetails
  - createNewChargeWithAccountToken
generated: '2026-08-09'
method: generated
---

# ChargeAfter ecommerce checkout financing

ChargeAfter is an orchestrator, not a lender. Approval, pricing and disclosure text are decisions made by
one of 40+ independent lenders in the merchant's configured network. Your job is to move the consumer
through the flow and render what the lender returns — never to interpret or restyle it.

## Before you start

- **Base URL.** `https://api.chargeafter.com` in production, `https://api-sandbox.ca-dev.co` in sandbox.
  There is no key prefix that distinguishes test from live credentials — the environment is carried by the
  base URL alone. Verify which one you are pointed at before any write.
- **Auth.** `Authorization: Bearer <PRIVATE_API_KEY>` on every server-side call, HTTPS only. The private key
  must never reach the browser; the browser uses the separate public key through the JavaScript SDK.
  See `authentication/chargeafter-authentication.yml`.
- **No operationIds.** ChargeAfter's published definitions declare none. The ids used below are the derived
  ids from `overlays/chargeafter-*-overlay.yaml`; the authoritative binding is the METHOD + path given with
  each step.

## Steps

1. **Create the session.** `POST /v3/session` (`createANewSession`) —
   `openapi/chargeafter-checkout-session-openapi.yml`. Send the cart, the consumer details you already hold,
   and the checkout details. The response is the session the rest of the flow hangs off.

2. **Read the offers.** `GET /v3/session/accounts` (`getLenderAccountsAndOffers`) —
   `openapi/chargeafter-checkout-session-accounts-openapi.yml`. Returns lender accounts and the offers each
   lender is willing to make.

   If you are on the older v2 surface instead, use `POST /v2/checkout/accounts`
   (`getAccountsWithOffersForConsumer`) in `openapi/chargeafter-distribution-api-openapi.yml`, which looks up
   an existing account and creates one when none exists. When you already hold an `applicationId`, use
   `POST /v2/checkout/applications/{applicationId}/accounts` instead so you do not start a second
   application.

3. **Render the offers exactly as returned.** Each `Offer` carries `Disclosure` / `OfferDisclosures`
   objects. This text is lender-regulated consumer-credit disclosure. ChargeAfter states that overriding or
   changing it "will create a compliance risk with your financing provider". Do not summarize, truncate,
   translate, or restyle it, and do not compute your own APR or monthly payment from the numbers.

4. **Poll the application if it is not immediately resolved.**
   `GET /v2/checkout/applications/{applicationId}/status` (`getStatusOfAnApplication`) —
   `openapi/chargeafter-checkout-application-status-openapi.yml`. The response carries the application
   status, the account details, and the confirmed offer.

   Prefer webhooks over polling where you can: `application.approved`, `application.declined`,
   `application.pending-lender`, `application.pending-consumer`, `application.pending-merchant` and
   `application.cancelled` all fire on this transition. See
   `asyncapi/chargeafter-notifications-webhooks.yml`. Note the webhooks carry no signature — authenticate
   deliveries some other way before acting on them.

5. **Bind the approved account to the cart.** `POST /v2/accounts/confirmation`
   (`createConfirmationWithApplicationAndCartDetails`) — `openapi/chargeafter-charge-api-openapi.yml`.
   This produces the confirmation that ties an approved account to this specific cart.

6. **Create the charge.** `POST /v2/accounts/charge` (`createNewChargeWithAccountToken`) —
   `openapi/chargeafter-charge-api-openapi.yml`. Creates the charge from the approved account using the
   `accountToken`.

## Rules an agent must follow

- **Never retry a charge blindly.** ChargeAfter defines no `Idempotency-Key` header on any of its 46
  published operations. If `POST /v2/accounts/charge` or `POST /v2/post-sale/charges` times out, you do not
  know whether it landed. Resolve state first with `GET /v2/post-sale/orders/{orderId}` (`getChargeId`) or
  `GET /v2/post-sale/transactions/lookup` (`getTransactionByMerchantTransactionId`) using your own
  `merchantOrderId` / `merchantTransactionId`, and only then decide whether to re-issue.
- **A confirmation is single-use.** Error `2002` — "charge already has assigned confirmation" — means the
  charge already exists. Treat it as a success signal that you must reconcile, not as a failure to retry.
- **Read errors from the envelope.** Failures return
  `{ requestId, errors: [ { code, description } ] }` on 400 and
  `{ requestId, message, errors: [ { field, validationState, error } ] }` on 422. Not RFC 9457. Capture
  `requestId` on every failure — it is the only ChargeAfter-side trace handle, and it is only returned on
  errors. Codes are in `errors/chargeafter-error-codes.yml`.
- **Do not assume pagination or rate-limit headers.** Neither exists in the contract. No operation declares
  a 429, so back off on your own schedule rather than reading a `Retry-After` that will not be there.
- **Handle a decline as a lender outcome, not an API error.** A declined application returns successfully;
  the decline arrives via `application.declined` or in the application status, not as a 4xx.
