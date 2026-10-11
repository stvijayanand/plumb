# ACH return codes, timelines and auth/capture mapping

Research for ticket #11 (feeds #12, the payment state machine for card and ACH). Vocabulary follows `GLOSSARY.md`: *payer*, *merchant*, *payout*, *payout return*, *payouts in transit*, *unsettled balance*, *available balance*, *pending entry*, *posted entry*.

Researched 2026-10-10. The full *Nacha Operating Rules* (Appendix Four lists every return code and its timeframe) are a paid publication. The claims below come from Nacha's public rule pages, the Federal Reserve's FedACH pages, and Stripe's docs. Claims marked **[secondary]** or **[inferred]** are not confirmed by a primary source.

## 1. ACH debit from the originator's side

### There is no authorization step

- An ACH debit is a one-way instruction. The originator (through its bank, the ODFI) pushes the entry into the network. The payer's bank (the RDFI) posts it on the settlement date, or returns it. Nothing like a card authorization exists: no real-time approval, no hold on funds, and no balance check at the network level. Stripe lists ACH Direct Debit as supporting **no** authorization features (extended, incremental or re-authorization) and **no** manual, partial or multi-capture. [Stripe ACH Direct Debit, Capabilities](https://docs.stripe.com/payments/ach-direct-debit#capabilities)
- What ACH has in place of authorization is the **mandate**: the payer's prior permission to debit their account. Nacha requires it, and the originator must keep it and produce it on request. Stripe records the business name, one-time vs recurring, and acceptance date, IP and user agent. A recurring debit needs at least 7 calendar days' notice before its timing changes. [Stripe, Mandates](https://docs.stripe.com/payments/ach-direct-debit#mandates)
- **Account validation** stands in for a "card check". For WEB (internet-initiated) debits, Nacha requires the originator to validate, by a commercially reasonable method, that the account is open and accepts ACH entries before its first use and whenever the account number changes. Ownership checks are not required. [Nacha, Supplementing Fraud Detection Standards for WEB Debits](https://www.nacha.org/rules/supplementing-fraud-detection-standards-web-debits); [Nacha Account Validation FAQ](https://www.nacha.org/system/files/2023-04/Account-Validation-FAQs-Oct-19-2020.pdf). Stripe meets this with Financial Connections, which verifies instantly, or with microdeposits, which take 1–2 business days to appear and expire after 10 days. [Stripe, accept a payment (Direct API)](https://docs.stripe.com/payments/ach-direct-debit/accept-a-payment?payment-ui=direct-api)
- The only "balance check" is optional and outside the network. With balance permissions, Stripe Financial Connections can read the payer's balance before the debit is confirmed. It is advisory and reserves nothing. [same page](https://docs.stripe.com/payments/ach-direct-debit/accept-a-payment?payment-ui=direct-api)

### Settlement mechanics (network level)

- FedACH settles forward entries in fixed windows. There are three same-day windows, settling at 1:00, 5:00 and 6:00 p.m. ET, and future-dated (next-day) entries settle at 8:30 a.m. ET on the settlement date. [FedACH Processing Schedule](https://www.frbservices.org/resources/resource-centers/same-day-ach/fedach-processing-schedule.html)
- For credits, the RDFI must make funds available by 1:30 p.m. local time (first same-day window), 5:00 p.m. (second), or end of processing day (third). A non-same-day credit is due by 9:00 a.m. on the settlement date if received by 5:00 p.m. the day before. [Nacha, ACH Schedules and Funds Availability](https://www.nacha.org/system/files/2021-03/SDA_Schedules_and_Funds_Availability.pdf); [Nacha, Providing Faster Funds Availability](https://www.nacha.org/rules/providing-faster-funds-availability)
- **Interbank settlement is not finality.** Money moves between banks on the settlement date, but the RDFI can still return the entry afterwards, within the windows below. "Settled" in ACH therefore means *settled but returnable*.

## 2. Return codes and timelines

All return clocks run from the **settlement date** of the original entry.

| Code | Meaning | Return window | Applies to | Source |
|---|---|---|---|---|
| R01 | Insufficient funds | 2 banking days (standard) | Debits | Standard window: [Nacha R17 page](https://www.nacha.org/rules/return-questionable-transaction) states "the standard 2-day return period"; code meaning **[secondary]** ([Plaid](https://plaid.com/en-eu/resources/ach/ach-return/)) |
| R02 | Account closed | 2 banking days | Debits and credits | Nacha groups R02/R03/R04 as "administrative" returns ([Nacha risk topics](https://www.nacha.org/rules/ach-network-risk-and-enforcement-topics)); window **[secondary]** |
| R03 | No account / unable to locate | 2 banking days | Debits and credits | as R02 |
| R04 | Invalid account number | 2 banking days | Debits and credits | as R02 |
| R09 | Uncollected funds | 2 banking days | Debits | **[secondary]** ([Plaid](https://plaid.com/en-eu/resources/ach/ach-return/)) |
| R10 | Payer says originator not known or not authorized | **60 calendar days** (consumer account), needs a Written Statement of Unauthorized Debit (WSUD) | Consumer debits | [Nacha, Differentiating Unauthorized Return Reasons](https://www.nacha.org/rules/differentiating-unauthorized-return-reasons); [Nacha, Limitation on Warranty Claims](https://www.nacha.org/rules/limitation-warranty-claims) |
| R11 | Entry not per the authorization's terms (wrong amount or date, improper reinitiation) | **60 calendar days**, WSUD | Consumer debits | [Nacha, Differentiating Unauthorized Return Reasons](https://www.nacha.org/rules/differentiating-unauthorized-return-reasons) |
| R05, R07, R51 | Other unauthorized / authorization revoked | 60 days (consumer) | Consumer debits | Counted as "unauthorized" by Nacha ([risk topics](https://www.nacha.org/rules/ach-network-risk-and-enforcement-topics)); 60-day window **[secondary]** |
| R29 | Corporate payer says not authorized | **2 banking days** | Non-consumer (business) debits | Nacha: unauthorized returns to non-consumer accounts are due "by the opening of business on the second banking day following the settlement date" ([Limitation on Warranty Claims](https://www.nacha.org/rules/limitation-warranty-claims)); code R29 counted as unauthorized ([risk topics](https://www.nacha.org/rules/ach-network-risk-and-enforcement-topics)) |
| R17 | Questionable / suspected fraud (false pretenses) | 2 banking days | Any | [Nacha risk management topics, Oct 2024](https://www.nacha.org/rules/risk-management-topics-october-1-2024) |
| R06 | Returned at ODFI's request | At RDFI discretion; RDFI must answer within 10 banking days | Any | [same](https://www.nacha.org/rules/risk-management-topics-october-1-2024) |
| R16, R20, R23 | Account frozen / non-transaction account / credit refused by receiver | 2 banking days | Typical **payout** returns | **[secondary]** ([Oracle](https://docs.oracle.com/cd/E78131_01/books/eBillingSDK/eBillingSDK_PaymentDesign6.html), [Helix](https://docs.helix.q2.com/docs/nacha-return-codes)) |

Precise timing details:

- **Consumer extended return.** The return is due "by the opening of business on the banking day following the 60th calendar day following the settlement date". Since October 2024, the RDFI must also send it by the sixth banking day after it finishes reviewing the signed WSUD. [Nacha, Limitation on Warranty Claims](https://www.nacha.org/rules/limitation-warranty-claims); [Nacha risk management topics](https://www.nacha.org/rules/risk-management-topics-october-1-2024)
- **Beyond the return windows.** For an unauthorized entry, the RDFI can still make a *warranty claim* (outside the return mechanism) against the ODFI: up to 1 year for non-consumer accounts, and 95 days up to 2 years for consumer accounts. [Nacha, Limitation on Warranty Claims](https://www.nacha.org/rules/limitation-warranty-claims). Stripe calls such out-of-window returns *late returns*, decided at the banks' discretion. [Stripe, Disputes](https://docs.stripe.com/payments/ach-direct-debit#disputes)
- **Reinitiation.** An originator can retry R01/R09 debits, at most 2 times, with the same Company Name, Company ID and Amount, and "RETRY PYMT" as the entry description. An entry returned as unauthorized must **not** be reinitiated without a new authorization. After an R11, a corrected entry may be sent within 60 days of the return without re-authorization. [Nacha risk and enforcement topics](https://www.nacha.org/rules/ach-network-risk-and-enforcement-topics); [Nacha Minor Rules Topics](https://www.nacha.org/rules/minor-rules-topics-1); [Nacha, Differentiating Unauthorized Return Reasons](https://www.nacha.org/rules/differentiating-unauthorized-return-reasons). The 180-day reinitiation window often quoted is **[secondary/unconfirmed]**. Stripe Billing retries an insufficient-funds failure at most 2 times within 40 days. [Stripe, Billing retries](https://docs.stripe.com/payments/ach-direct-debit#billing-retries)
- **Return-rate thresholds the originator is monitored on.** Unauthorized returns (R05, R07, R10, R29, R51; R11 is also included) have a 0.5% threshold. Administrative returns (R02, R03, R04) have a 3% level, and all debit returns a 15% level. [Nacha risk and enforcement topics](https://www.nacha.org/rules/ach-network-risk-and-enforcement-topics); [Nacha, Differentiating Unauthorized Return Reasons](https://www.nacha.org/rules/differentiating-unauthorized-return-reasons). These are useful inputs for the merchant risk policy.
- **Reversals** of erroneous entries by the originator must reach the RDFI within 5 banking days of settlement. [Nacha, Reversals and Enforcement](https://www.nacha.org/rules/reversals-and-enforcement)

**Why this matters for B2B.** Plumb's payers are businesses, so their debits are usually to non-consumer accounts, where even an *unauthorized* return must come within 2 banking days (R29). The 60-day tail (R10/R11) applies only when the payer's account is a consumer account. That happens with sole proprietors, and the RDFI, not the originator, decides which kind of account it is. Stripe states the same split: "up to 60 calendar days after a debit on a personal account, or up to two business days for a business account". [Stripe, Disputes](https://docs.stripe.com/payments/ach-direct-debit#disputes)

## 3. How Stripe exposes ACH debit states

- PaymentIntent: `requires_payment_method` → (`requires_action` while microdeposit verification is pending) → `requires_confirmation` (mandate is shown) → **`processing`** on confirm → **`succeeded`** or back to **`requires_payment_method`** on failure. Events: `payment_intent.processing`, `payment_intent.succeeded`, `payment_intent.payment_failed`. [Stripe, accept a payment (Direct API)](https://docs.stripe.com/payments/ach-direct-debit/accept-a-payment?payment-ui=direct-api)
- Charge: `charge.pending` → `charge.succeeded` ("the funds are available in your balance") or `charge.failed`, with `failure_code`. Once `charge.updated` fires, the payment can no longer be cancelled. `payment_method_details.us_bank_account.expected_debit_date` gives an estimated debit date. [same](https://docs.stripe.com/payments/ach-direct-debit/accept-a-payment?payment-ui=direct-api); [Stripe ACH timing](https://docs.stripe.com/payments/ach-direct-debit#timing)
- **Timing:** `succeeded` is reported within **up to 4 business days** (standard settlement, T+4), or 2 business days with faster settlement (T+2, for eligible accounts). At that point Stripe has *settled* the funds into the Stripe balance. [Stripe ACH timing](https://docs.stripe.com/payments/ach-direct-debit#timing). Stripe's T+4 lets the 2-banking-day return window pass before it reports `succeeded`, so most R01/R02/R03/R04 returns surface as `payment_failed` / `charge.failed`, not as a reversal after success **[inferred from the timelines]**.
- **Returns after `succeeded`** come back as **disputes**:
  - Unauthorized (R10/R11/R29-style): `charge.dispute.created`. These are final and cannot be contested through the network, and the mandate becomes invalid.
  - A late administrative failure becomes a dispute with `reason` `insufficient_funds`, `incorrect_account_details` or `bank_cannot_process`, and Stripe charges a failure fee.
  - Funds already made available are removed from the balance immediately.
  - A bank's proof-of-authorization request surfaces as a dispute with status `warning_needs_response`.

  [Stripe, Transaction failures and Disputes](https://docs.stripe.com/payments/ach-direct-debit#transaction-failures)
- Refunds are separate ACH credits, not reversals. They allow full refunds only, within 180 days, take up to 3 business days, and can themselves fail (`refund.failed`, with the funds going back to the balance). [Stripe, Refunds](https://docs.stripe.com/payments/ach-direct-debit#refunds)
- Test bank accounts simulate each outcome: `accountClosed`, `noAccount`, `insufficientFunds`, `debitNotAuthorized`, `dispute`, and `processing` (stays processing indefinitely). Test-mode settlement is instant. [Stripe, Test account numbers](https://docs.stripe.com/payments/ach-direct-debit/accept-a-payment?payment-ui=direct-api#test-account-numbers). These are a good template for the simulated rail.

## 4. ACH credits for payouts and payout returns

- A payout is an ACH **credit** that the originator pushes. No mandate from the receiver is needed in the debit sense, and the receiving bank must make the funds available on the settlement date (see section 1).
- A payout can still be **returned** by the receiving bank, for example closed, nonexistent or invalid account (R02/R03/R04), frozen account (R16), non-transaction account (R20), or credit refused (R23). These come back within the same ~2-banking-day window **[R16/R20/R23 timing secondary]**. R01 does not apply to credits **[inferred]**.
- Stripe Payout `status`: `pending` (not yet submitted) → `in_transit` → `paid`, or `failed` / `canceled` "within 5 business days". "Some payouts that fail might initially show as `paid`, then change to `failed`." On failure, `failure_balance_transaction` returns the funds to the balance. `failure_code` values include `account_closed`, `no_account`, `invalid_account_number`, `account_frozen`, `could_not_process` and `declined`. [Stripe, The Payout object](https://docs.stripe.com/api/payouts/object)
- This matches the glossary's *payout return*, and the lifecycle settled in map #2 (scheduled → initiated → sent → paid, with a returned branch). Note that **`paid` is not terminal**: a payout return can arrive after `paid`.

## 5. Implications for the state machine (#12)

1. **ACH has no authorize/capture split.** Map ACH "confirm with mandate" to a single *submit* transition that is both. There is no `authorized` state, no void after submit (only cancel before the PSP locks the charge), and no partial capture. Card states stay as they are. ACH uses a separate path: `created → mandate_accepted → submitted (processing) → succeeded | failed`.
2. **Prerequisites before submit:** a mandate exists and the account is verified (instant or microdeposit). Microdeposit verification is an asynchronous wait state (`requires_action`) that can time out after 10 days.
3. **`failed` before `succeeded`** covers the 2-banking-day returns (R01/R02/R03/R04/R09/R29) and verification failures. Store the return code. R01/R09 failures are retryable (max 2 reinitiations, "RETRY PYMT"). Unauthorized codes are **not** retryable without a new mandate, and administrative codes need new account details.
4. **`succeeded` is not terminal.** Add a `returned` (late-return or dispute) transition out of `succeeded`. It stays open for 60 days plus banking-day slack on consumer accounts, and in theory warranty claims run beyond that. The mandate is invalidated on an unauthorized return.
5. **Payouts:** `sent`/`paid` → `returned` stays possible for several business days after `paid` (Stripe: up to 5 business days). Reconciliation of payouts in transit must tolerate a `paid` that flips.
6. **Simulated rail:** reuse Stripe's test-account behaviours (closed, no account, insufficient funds, not authorized, dispute, stuck processing). Model settlement on banking days, using the FedACH windows.

## 6. Implications for the ledger (unsettled → available)

- **No pending entry at submit, or a pending entry with different meaning.** Card authorization writes a pending entry because the issuer really holds funds. ACH reserves nothing, so a pending entry at submit would only record *expected* funds. Recommendation: record nothing posted until the PSP reports `succeeded`. If #12 wants visibility, write a pending entry at submit and close it with nothing posted on failure, as for a card void.
- **On `succeeded`,** post the capture into the merchant's **unsettled balance**, as for a card capture. Stripe's `succeeded` already means the funds sit in the PSP balance. They move to **available** when the PSP's payout of those funds lands in the FBO account (the normal settlement step). By then the 2-banking-day return window has passed, because Stripe waits T+2/T+4.
- **The residual 60-day tail** for consumer-account unauthorized returns (R10/R11), and late returns, should be covered by the **risk reserve** and the merchant risk policy (return-rate inputs that mirror Nacha's 0.5% / 3% / 15% levels), not by holding funds in unsettled for 60 days. A return after funds became available debits merchant payable, which the glossary already allows to go negative.
- **Payout return:** reverse *payouts in transit* (or the paid payout) back into merchant payable (available) as posted entries. Consider whether a payout hold applies until new bank details exist (codes R02/R03/R04/R16/R20/R23).

## Open questions / gaps

- The exact Appendix Four timeframes for R01–R04, R09, R16, R20 and R23 rely on secondary sources. Nacha's public pages confirm only "the standard 2-day return period" generically.
- The 180-day reinitiation window is unconfirmed by a primary source.
- Whether Plumb's simulated rail should model the consumer-vs-business distinction (R10 at 60 days vs R29 at 2 days). The recommendation is yes, as an account-type flag on the payer's bank account, since B2B with sole proprietors hits both.
