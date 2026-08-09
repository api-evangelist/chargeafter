---
name: ChargeAfter funding reconciliation
description: >-
  Reconcile merchant settlements against lender funding using ChargeAfter's funding report — pull the
  funding records for a period, match them to charges by merchant transaction id, and find the settled
  charges that were never funded.
api: openapi/chargeafter-funding-api-openapi.yml
apis:
  - openapi/chargeafter-funding-api-openapi.yml
  - openapi/chargeafter-charge-api-openapi.yml
operations:
  - getFundingReportNew
  - getTransactionByMerchantTransactionId
  - getDetailsOfACharge
generated: '2026-08-09'
method: generated
---

# ChargeAfter funding reconciliation

A settled charge is not money in the bank. ChargeAfter orchestrates lenders; the lender funds the merchant
on its own schedule. The funding report is the only surface that tells you what actually moved.

## Steps

1. **Pull the funding records.** `GET /v2/post-sale/fundings/report` (`getFundingReportNew`) —
   `openapi/chargeafter-funding-api-openapi.yml`. Returns funding records for the supplied filters.

   Fields the changelog records as having been added over time, and which you should expect and use:
   - `merchantTransactionId` — the merchant-side key you match on
   - `ProductType` — the financing product the lender wrote
   - `PromoCode` — the promotion applied
   - reconciliation status values
   - `showMissing` — the parameter that surfaces records expected but not present

2. **Set the window deliberately.** The endpoint takes date and status filters and returns the whole
   filtered set — there is no pagination anywhere in the ChargeAfter API. Pull a day or a settlement period
   at a time rather than a quarter, or you will ask for a response you cannot stream.

3. **Match to your ledger on `merchantTransactionId`.** This is your identifier, set at charge creation. If
   it was not set, you have no clean join key; fall back to
   `GET /v2/post-sale/transactions/lookup` (`getTransactionByMerchantTransactionId`) and
   `GET /v2/post-sale/charges/{chargeid}` (`getDetailsOfACharge`) in
   `openapi/chargeafter-charge-api-openapi.yml` to walk it back.

4. **Use `showMissing` to find the gap.** The records you care about most are the settled charges with no
   corresponding funding record. That is the reconciliation exception queue.

5. **Escalate exceptions to ChargeAfter with a `requestId`** where you have one, and with the
   `merchantTransactionId` and charge id where you do not.

## Rules an agent must follow

- **Read-only.** Nothing in this skill writes. If a reconciliation discrepancy looks like it needs a refund
  or a void, hand it to a human — do not resolve a funding gap by moving money.
- **Never treat a settle response as proof of funding.** They are different events, days apart.
- **Do not paginate.** There is no page/limit/cursor parameter. Narrow by date and status instead.
- **Do not assume a rate limit signal.** No 429 is declared on this operation. If you are sweeping many
  periods, pace yourself; you will get no `Retry-After` to guide you.
- **Funding data is financial PII.** Do not export it into general-purpose logs or third-party analytics.
