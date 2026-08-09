---
name: ChargeAfter OmniLink in-store and call-center financing
description: >-
  Generate a single-use consumer link that launches the ChargeAfter Apply or Checkout experience on the
  consumer's own device, for in-store and call-center sales where the consumer is not at a browser you
  control.
api: openapi/chargeafter-omni-link-openapi.yml
apis:
  - openapi/chargeafter-omni-link-openapi.yml
  - openapi/chargeafter-checkout-application-status-openapi.yml
  - openapi/chargeafter-checkout-application-token-openapi.yml
operations:
  - generateConsumerApplicationLink
  - generateConsumerCheckoutLink
  - getLinkData
  - getStatusOfAnApplication
  - getInteractiveToken
generated: '2026-08-09'
method: generated
---

# ChargeAfter OmniLink in-store and call-center financing

OmniLink exists for the case where the sale is happening in a store or on a phone call and the consumer
should complete the credit application on their own device — so that the associate never handles the
consumer's SSN, income, or identity data.

Base URL `https://api.chargeafter.com` (sandbox `https://api-sandbox.ca-dev.co`), auth
`Authorization: Bearer <PRIVATE_API_KEY>`.

## Steps

1. **Choose the flow.**
   - `POST /v2/omni-link/consumer/application` (`generateConsumerApplicationLink`) — generates a URL that
     launches the **Apply** experience. Use when you are prequalifying or approving a consumer ahead of a
     specific cart.
   - `POST /v2/omni-link/consumer/checkout` (`generateConsumerCheckoutLink`) — generates a URL that launches
     the **Checkout** experience against a known cart.

   Both are in `openapi/chargeafter-omni-link-openapi.yml`.

2. **Deliver the link to the consumer directly.** The link carries the consumer context. Send it to the
   consumer's own phone or email; do not open it on a shared store terminal and do not read it aloud.

3. **Track the application.** `application.created` fires when the session begins — the documentation calls
   this out specifically as the way to track applications initiated through the OmniLink API. Then
   `application.prequalified`, `application.approved`, `application.pending-*`, `application.declined` or
   `application.cancelled` follow. See `asyncapi/chargeafter-notifications-webhooks.yml`.

   If you cannot receive webhooks, poll `GET /v2/checkout/applications/{applicationId}/status`
   (`getStatusOfAnApplication`) in `openapi/chargeafter-checkout-application-status-openapi.yml`.

4. **Read link state when you need it.** `GET /v2/omni-link/consumer/{linkId}` (`getLinkData`) returns the
   data for the consumer-associated context behind a generated link.

5. **Resume an interrupted checkout.** `GET /v2/checkout/applications/{applicationId}/token`
   (`getInteractiveToken`) in `openapi/chargeafter-checkout-application-token-openapi.yml` returns a
   **time-limited** interactive session token that resumes the Checkout flow for the account associated with
   the application. It expires — fetch it at the moment you need it, never cache it, and never log it.

6. **Complete the sale** through the normal confirmation and charge path — see
   `skills/chargeafter-ecommerce-checkout-financing.md` from step 5.

## Rules an agent must follow

- **Links are consumer credentials.** A generated OmniLink launches a credit application bound to a specific
  consumer context. Treat the URL, the `linkId`, and any interactive token as secrets: no logging, no
  analytics, no CRM notes, no screenshots.
- **Do not generate a second link to "retry".** Link generation has no idempotency contract. Re-issuing
  creates a second application context and can produce duplicate applications against the same consumer.
  Read `getLinkData` or the application status first.
- **The associate must not fill in the consumer's data.** The entire point of the flow is that the consumer
  enters their own identity and income information. Never collect it on their behalf.
- **Two of the published OmniLink operations are not on the current definition.** An older published build
  of the same API (OmniLink 1.7.12, reachable through `/.well-known/api-catalog`) also declares
  `POST /v2/omni-link/consumer/modify-loan` and `POST /v2/omni-link/consumer/additional-action`. They are
  not in the definition linked from the docs navigation. Confirm with ChargeAfter before depending on them.
