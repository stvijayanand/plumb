# Stripe test-mode API shape for the PSP interface

Research for ticket #14 (map #2). It feeds #15, the ADR on one PSP interface with the simulator built first.

**Question.** What does Stripe's test-mode API look like for the operations Plumb's PSP interface needs? The operations are manual capture (authorize, then capture), partial capture, refunds, voids, ACH Direct Debit, webhooks, idempotency keys, and settlement and payout reports. Where would a generic PSP interface built for a simulator not fit Stripe?

**Sources.** Only first-party Stripe docs (docs.stripe.com), read 2026-10-10. Each claim cites its page. Lines marked *Inference* are my conclusions, not Stripe's statements.

## Short answer

- Stripe models a payment as one **PaymentIntent** with a status machine, not as separate authorize, capture and void calls. Authorize means create and confirm with `capture_method=manual`, which reaches `requires_capture`. Capture is `POST /capture` with an optional `amount_to_capture`. Void is `POST /cancel`. Refund is a separate **Refund** object with its own async lifecycle.
- **A partial capture is final.** Stripe releases the remainder automatically, and for most payments you can't capture a second time. This matches Plumb's rule that "a partial capture lets the remainder lapse".
- **ACH Direct Debit does not support manual capture, partial capture or partial refunds.** It goes straight from `processing` to `succeeded` or `requires_payment_method` over up to 4 business days. It can still fail after `succeeded`, which Stripe raises as a dispute. So one authorize-then-capture interface doesn't fit ACH.
- **Webhooks are at-least-once and unordered.** Duplicates happen and order isn't guaranteed. Stripe says to dedupe on event ID and not to use `created` for ordering. In a sandbox, retries run only 3 times over a few hours (3 days in live mode). The events API goes back 30 days.
- **Idempotency** uses the `Idempotency-Key` header on every POST. Keys are kept for at least 24 hours. Stripe saves the first response, *including 500s*, and returns it on replay. A 500 or a network timeout is an **unknown outcome**: reconcile by retrieving the object and reading webhooks, and don't retry with a new key.
- **Settlement shows up in balance transactions** (`status` pending→available, `available_on`, `fee`, `net`). Payouts are objects with `pending → in_transit → paid | failed | canceled` and can flip from `paid` to `failed`. The Reports API needs a **live-mode key**, so test mode can't use it. Use `GET /v1/balance_transactions?payout=` instead.
- **Test mode is not live mode.** Test ACH settles instantly. Test cards go to the pending balance, except a bypass-pending card. Rate limits are lower (25 rps in a sandbox against 100 rps live), and network responses are mocked. A simulator must not copy test-mode timing.

## 1. PaymentIntent lifecycle

Statuses are `requires_payment_method` → `requires_confirmation` → `requires_action` → `processing` | `requires_capture` → `succeeded` | `canceled`. "If the payment attempt fails (for example, due to a decline), the PaymentIntent's status returns to `requires_payment_method` so that the payment can be retried." A decline is therefore **not a terminal state** of the intent. Cancellation "can't be undone". [lifecycle]

With manual capture, "your PaymentIntent can instead move to `requires_capture`. In that case, attempting to capture the funds moves it to `processing` or `succeeded` depending on the payment method." [lifecycle]

## 2. Authorize, then capture (manual capture)

- Create with `capture_method=manual`, which can also be set per method with `payment_method_options[card][capture_method]=manual`. Confirm with `payment_method` and `confirm`. On success the status is `requires_capture` and `payment_intent.amount_capturable_updated` fires. [hold]
- Authorization windows for card-not-present payments: about 7 days for customer-initiated payments on all brands; for Visa merchant-initiated payments, 5 days ("4 days and 18 hours"). Extended authorization is available for eligible payments. [hold]
- **Expiry:** "If the authorization expires before you capture the funds, the funds are released and the payment status changes to `canceled`." The expiry time is on `charge.payment_method_details.card.capture_before`. [hold] This maps to Plumb's "void or expiry closes the pending entry with nothing posted". Plumb learns about an expiry from a webhook or a poll, not from its own call.
- Capture: `POST /v1/payment_intents/{id}/capture`, which "captures the total authorized amount by default". [hold]
- An auto-capture-before-expiry option (`automatic_delayed`) exists but is in **private preview**. [hold]

## 3. Partial capture

- Pass `amount_to_capture`. "A partial capture automatically releases the remaining amount." [hold]
- "You can only perform one capture on an authorized payment for most payments. If you partially capture a payment, you can't perform another capture for the difference." Multicapture and overcapture exist only for some card payments. [hold]
- ACH capabilities are listed as ✗ Manual capture, ✗ Partial capture, ✗ Multi-capture. [ach]

## 4. Void (cancellation)

- `POST /v1/payment_intents/{id}/cancel` takes an optional `cancellation_reason` ∈ {`duplicate`, `fraudulent`, `requested_by_customer`, `abandoned`}. [cancel]
- It is allowed from `requires_payment_method`, `requires_capture`, `requires_confirmation`, `requires_action`, and from `processing` only for US bank accounts (ACH) and other bank debits, "might fail due to a limited and varying cancellation time window". [cancel] [refunds] [lifecycle]
- "For PaymentIntents with a status of `requires_capture`, the remaining `amount_capturable` is automatically refunded." Cancellation returns an error "if the PaymentIntent is already canceled or isn't in a cancelable state". [cancel]
- An uncaptured PaymentIntent can't be refunded: "You must cancel the PaymentIntent." [refunds]

## 5. Refunds

- `POST /v1/refunds` takes `payment_intent` or `charge`, plus an optional `amount`, `reason` ∈ {`duplicate`, `fraudulent`, `requested_by_customer`} and `metadata`. Partial refunds are allowed "multiple times, until the entire charge has been refunded". Over-refunding returns an error. [refund-create]
- A refund is **asynchronous and can fail later**. Statuses include `pending`, `requires_action`, `succeeded`, `failed` and `canceled`. A failed refund puts the money back in the Stripe balance, "up to 30 days" later, and sets `failure_reason` and `failure_balance_transaction`. Events are `refund.created`, `refund.updated` and `refund.failed`. [refunds]
- Test card `4000000000005126` (`pm_card_refundFail`): the refund "begins as `succeeded`, then later transitions to `failed` and sends a `refund.failed` event". [testing] So **`succeeded` is not final for a refund.**
- Balance effect: "Refunds use your available Stripe balance … If your available balance doesn't cover the amount of the refund, Stripe holds the refund as pending for card transactions (refunds for other payment method types fail)." [refunds]
- ACH refunds: full refunds only, within 180 days, async for up to 3 business days, and they **can't be canceled**. Stripe also warns of a double-credit risk if a dispute and a refund overlap. [ach]

## 6. ACH Direct Debit (`us_bank_account`)

- It is a "delayed notification" payment method. Confirm leads to `processing` and then, up to 4 business days later (T+4, or T+2 for eligible accounts), to `succeeded` or a failure (`requires_payment_method` + `payment_intent.payment_failed`). [ach-accept] [ach]
- Before confirming, it needs a **mandate** (customer acceptance) and bank account **verification**: Financial Connections (instant) or microdeposits (`next_action.verify_with_microdeposits`, 1–2 business days, a 10-day timeout). [ach-accept]
- **It can fail after success.** "In rare situations, Stripe might receive an ACH failure from the bank after a PaymentIntent has transitioned to `succeeded`. If this happens, Stripe creates a dispute." A business-account payer can dispute within 2 business days, a personal account within 60 days. "If a payment fails after funds have been made available in your Stripe balance, Stripe immediately removes funds." [ach]
- Test bank accounts use routing `110000000`: `000123456789` succeeds, `000222222227` fails with insufficient funds, `000111111113` fails as account closed, `000555555559` triggers a dispute, and `000000000009` "stays in processing indefinitely" (for testing cancellation). Microdeposit test codes include `SM11AA`, which verifies. [ach-accept]
- **Test-mode timing:** "Test transactions settle instantly and are added to your available test balance. This behavior differs from live mode." [ach-accept]

## 7. Webhooks: delivery guarantees

- **Order:** "Stripe doesn't guarantee the delivery of events in the order that they're generated … Don't use `created` to determine event order." [webhooks]
- **Duplicates:** "Webhook endpoints might occasionally receive the same event more than once", so dedupe on event ID. Also, "in some cases, two separate Event objects are generated and sent": dedupe these on `data.object.id` + `event.type`. [webhooks]
- **Retries:** "up to three days with an exponential back off in live mode … We retry event deliveries created in a sandbox three times over the course of a few hours." [webhooks]
- **Recovery:** list events "going back up to 30 days". The `delivery_success=false` filter returns undelivered events. [events-list]
- **Security:** the `Stripe-Signature` header carries an HMAC-SHA256 over `t.payload`. SDKs use a default tolerance of 5 minutes. Return `2xx` fast and process asynchronously. [webhooks]
- **Snapshot vs thin:** a snapshot event carries the object as of the event, a thin event carries only a reference that you fetch. [webhooks] *Inference:* because events arrive out of order, a handler should treat any event as a hint and re-fetch the current object before it changes ledger state.

## 8. Idempotency and error model

- The `Idempotency-Key` header goes on POST (all v1 POSTs accept it). Keys are up to 255 characters and kept for at least 24 hours. "Retries with the same key return the saved response, including `500` errors." Reusing a key with different parameters returns an error. If validation fails or a concurrent request conflicts, nothing is saved and you may retry. [idem]
- **409** means a request with the same key is still running: wait and retry with the same key. A new key "can start a second operation". [errors]
- **Network error or timeout:** "Restoring connectivity doesn't tell you whether the original request succeeded." Retry with the same key and parameters within 24 hours. After that, "don't retry with either the old key or a new one", and reconcile instead. [errors]
- **5xx:** "Treat the outcome … as indeterminate: the operation might have produced side effects." Stripe may reconcile during incident recovery and send webhooks "for any objects created during reconciliation", so handlers must accept objects they never saw in an API response. Put a local ID in `metadata` to match them. [errors]
- `Idempotent-Replayed: true` marks a replayed response. `Stripe-Should-Retry` gives an explicit retry hint. [errors]
- **429** is a rate limit, or `lock_timeout` when another request holds the object. The request was not processed, so retry with backoff. Serialize mutations on the same object. [rate-limits]
- **402** means a valid request failed, for example a card decline. [errors]

## 9. Settlement, balance and payouts

- **BalanceTransaction** has `amount`, `fee`, `fee_details`, `net`, `status` ∈ {`pending`, `available`}, `available_on`, `type` (e.g. `charge`, `refund`, `refund_failure`, `payment_failure_refund`, `payout`, `payout_failure`, `stripe_fee`, `adjustment`), `reporting_category` and `source`. [bt]
- US card settlement starts at 2 business days. "Settlement timing is the time until funds become available, which is separate from the payout schedule." [payouts]
- **Payout** statuses: "`pending` until it's submitted to the bank, when it becomes `in_transit`. The status changes to `paid` if the transaction succeeds, or to `failed` or `canceled` (within 5 business days). Some payouts that fail might initially show as `paid`, then change to `failed`." `failure_balance_transaction` reverses the payout. [payout-obj]
- Listing the transactions inside a payout (`GET /v1/balance_transactions?payout=po_…`) works only for **automatic** payouts (`reconciliation_status`). "Reconciliation for manual payouts … You're responsible for reconciling." [recon] [payout-obj]
- **Reports API** (`/v1/reporting/report_runs`, e.g. `balance_change_from_activity.itemized.3` and the payout reconciliation report types) is asynchronous: it emits `reporting.report_run.succeeded` with a file URL. Stripe's example notes "A live-mode API key is required", and "if you use sandboxes, you won't receive `reporting.report_type.updated` events". [reports] **Plan to use the BalanceTransactions API in test mode, not report files.**
- Test card `4000000000000077` (`pm_card_bypassPending`) sends funds straight to the available balance. Other test cards go to the pending balance. [testing]

## 10. Test-mode limits worth knowing

- Global rate limit: 25 rps in a sandbox against 100 rps live. Stripe discourages load-testing against a sandbox because it mocks network calls ("significantly different latency profiles") and recommends mocking Stripe for load tests. [rate-limits]
- Card state carries over between tests, so use a separate test card for each scenario. [testing]
- Webhooks are tested with real sandbox actions or `stripe trigger` / `stripe listen --forward-to`. [webhooks]

## Where a simulator-first generic interface would not fit Stripe

1. **Synchronous result vs status machine.** A simulator naturally returns `APPROVED | DECLINED` from `authorize()`. Stripe returns an intent in one of several statuses, including `requires_action` (3DS, microdeposits) and `processing` (ACH). A decline sends the intent back to `requires_payment_method` rather than ending it. *Inference:* the interface needs a non-terminal **Pending/Processing** outcome, completed later by an event.
2. **ACH has no auth/capture split.** ACH has no manual capture, no partial capture and no partial refund, and it takes days. Its failures can arrive after `succeeded` as disputes. An interface that assumes every payment is "authorize, then capture" breaks for ACH. *Inference:* model *capability per payment method* (supports separate capture, partial capture or partial refund) or keep two flows, card holds and bank debits.
3. **Expiry is pushed by the PSP.** Stripe cancels an expired authorization on its own. Plumb finds out from a webhook or a poll, so the simulator must emit expiry as an async event, not just reject a late capture.
4. **Refund outcomes are async and reversible.** A refund can be `pending` or `requires_action`, and can even go `succeeded → failed` up to 30 days later. A synchronous `refund() → OK` is wrong.
5. **Payout `paid` is not final either.** A payout can move `paid → failed`. Plumb's own payout flow already has a returned branch. The Stripe payout (Stripe balance → FBO bank) is a different leg and needs the same shape.
6. **Unknown outcomes are first-class.** Timeouts and 5xx are "indeterminate", and Stripe may create the object later through reconciliation. A simulator that throws an exception and assumes nothing happened would hide the hardest case.
7. **Reports in test mode.** A simulator can emit a settlement file. In Stripe test mode, report runs need live keys, so reconciliation data has to come from BalanceTransactions or events.
8. **Test-mode timing misleads.** Test ACH settles instantly, while live ACH takes T+4 with a 60-day return tail. The simulator, not Stripe test mode, should be the source of realistic delays and failure injection.
9. **Fees and net amounts.** Stripe reports `fee` and `net` per balance transaction, and fees aren't returned on refunds. [refunds] A simulator that settles gross amounts would hide a ledger line Plumb needs (PSP fee expense).
10. **Party model.** Stripe sees one account, the platform's. Plumb's merchant split and FBO-to-merchant payouts sit outside Stripe unless Connect is used. *Inference:* keep merchant attribution in Plumb, and pass the Plumb payment ID in Stripe `metadata`.

## Implications for #15 (the PSP interface ADR)

*Inference throughout, grounded in the findings above.*

**Operations**, sketched in Plumb terms:

- `createPayment(paymentId, amount, method, captureMode, idempotencyKey) → PspPaymentRef`. Create and persist the PSP reference *before* confirming, so a later timeout still leaves a handle to retrieve. For Stripe this is create the PaymentIntent with `metadata.plumb_payment_id`, then confirm it.
- `authorize(ref)` / `confirm(ref) → Outcome`
- `capture(ref, amount ≤ authorized, idempotencyKey) → Outcome`, with a single capture only. The remainder lapses.
- `void(ref, reason, idempotencyKey) → Outcome`. It is valid only from authorized states, or from ACH `processing` within a window.
- `refund(ref, amount, reason, refundId, idempotencyKey) → RefundOutcome`, with its own lifecycle.
- `retrieve(ref)` / `retrieveRefund(id)`, which is mandatory for reconciling unknown outcomes.
- `capabilities(method)` reports whether a method supports separate capture, partial capture and partial refund.
- An inbound event stream (`PspEvent`: payment authorized/succeeded/failed/expired/canceled, refund succeeded/failed, dispute opened, balance transaction available, payout paid/failed) delivered **at-least-once and unordered**.
- `listBalanceTransactions(since | payoutRef)` for settlement and fee data.

**Error and outcome model:**

- `Succeeded(state)`, plus `Pending(state)` for async outcomes (ACH processing, refund pending, payout in transit).
- `Declined(code)` is a business result, not an exception. The payment can be retried with a new method.
- `Rejected(invalid request / invalid state)` is safe and nothing happened (4xx before execution, wrong state).
- `RetryableNotExecuted` (429 rate limit or `lock_timeout`, 409 in-flight) means retry with the **same** key.
- `Unknown` (timeout, connection reset, 5xx) means nothing is known. Retry with the same key for up to 24 hours, otherwise run `retrieve` or wait for an event. Never use a new key and never write a posted entry until it is resolved. The ledger keeps the pending state, and a reconciliation job resolves it.
- Idempotency keys are Plumb-generated, stored per *operation* (not per HTTP call), and reused across process restarts. Stripe's 24-hour key window bounds how long a retry is safe.

**Simulator requirements implied by Stripe:** inject duplicate and out-of-order events, timeout-after-success, 5xx-with-side-effect, auth expiry, refund `succeeded → failed`, ACH return after `succeeded`, and payout `paid → failed`. If the simulator can't produce these, the interface will be shaped around a PSP friendlier than Stripe.

## Sources

- [hold] Place a hold on a payment method: https://docs.stripe.com/payments/place-a-hold-on-a-payment-method
- [lifecycle] How Payment Intents work: https://docs.stripe.com/payments/paymentintents/lifecycle
- [cancel] Cancel a PaymentIntent (API): https://docs.stripe.com/api/payment_intents/cancel
- [refund-create] Create a refund (API): https://docs.stripe.com/api/refunds/create
- [refunds] Refund and cancel payments: https://docs.stripe.com/refunds
- [ach] ACH Direct Debit: https://docs.stripe.com/payments/ach-direct-debit
- [ach-accept] Accept an ACH Direct Debit payment (Direct API): https://docs.stripe.com/payments/ach-direct-debit/accept-a-payment?payment-ui=direct-api
- [webhooks] Receive Stripe events: https://docs.stripe.com/webhooks
- [events-list] List all events (API): https://docs.stripe.com/api/events/list
- [idem] Idempotent requests (API): https://docs.stripe.com/api/idempotent_requests
- [errors] Advanced error handling: https://docs.stripe.com/error-low-level
- [rate-limits] Rate limits: https://docs.stripe.com/rate-limits
- [bt] Balance Transaction object (API): https://docs.stripe.com/api/balance_transactions/object
- [payout-obj] Payout object (API): https://docs.stripe.com/api/payouts/object
- [payouts] Payouts: https://docs.stripe.com/payouts
- [recon] Payout reconciliation: https://docs.stripe.com/payouts/reconciliation
- [reports] Reports API: https://docs.stripe.com/reports/api
- [testing] Testing: https://docs.stripe.com/testing

**Not verified:** whether test mode lets you force an authorization to expire early. I found no such control in the pages above, so assume you would need to wait out the real window or simulate expiry.
