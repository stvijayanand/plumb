---
status: accepted
---

# A merchant shortfall becomes due from merchant, covered by the platform and recovered from future credits

A refund or a payer's ACH return can take more from a merchant than the platform still holds for it, typically after a payout. Plumb always posts the reversal, takes what it can from the merchant's own money, and records the rest as **due from merchant**: a per-merchant claim on the platform's books, outside the FBO account. The platform tops up the FBO account from operating cash for that amount, as [ADR 0005](0005-fbo-funds-as-agent-of-payee.md) requires (no merchant is funded by another merchant's money). It then recovers the money from the merchant's later credits, then by ACH debit, and writes off what it can't recover. We chose a separate account over letting available go negative because it keeps every merchant payable account non-negative, keeps the FBO identity free of special cases, and makes the platform's exposure a single balance.

## The policy

1. **Reversals are never refused for lack of funds.** Refunds and payer ACH returns always post. ACH returns can't be refused, so the shortfall path exists anyway.
2. **Waterfall.** The chart-of-accounts rule picks the first account: unsettled for an unsettled payment, available for a settled one. If that account runs out, the reversal takes from **available → unsettled → risk reserve**, and the remainder goes to **due from merchant**. Payouts in transit are never touched; that money has already left. Taking from unsettled offsets against money the PSP already owes this merchant.
3. **Top-up.** The entry that creates a due-from-merchant balance also posts Dr FBO cash / Cr operating cash for the same amount. One daily funding transfer moves the real money between operating cash and FBO: the day's top-ups minus the day's recoveries, before the sponsor bank's balance-file cutoff.
4. **Recovery from credits.** Every credit to a merchant pays down due from merchant before anything reaches available: settlements (unsettled → available), reserve releases and payout returns. A merchant that owes money can't be paid out, with no hold needed. The reserve isn't topped up after the waterfall uses it; holdback carries on at the policy's percentage on new captures.
5. **ACH debit.** If a balance is still due 5 business days (configurable) after the cover, the payouts scheduler debits the whole balance from the merchant's own bank account on file, authorized in the merchant agreement. It's a bank-only rail, consistent with ADR 0005. Nothing posts at initiation. When the funds land: Dr operating cash / Cr due from merchant (the money arrives in FBO and leaves in the daily funding transfer). A later return reverses that posting. R01 and R09 are retried once, after 2 business days. Any other return code goes straight to write-off.
6. **Write-off.** When the debit fails for good, or the balance is more than 60 days old, whichever comes first: Dr merchant losses / Cr due from merchant, tracked per merchant. The claim stands: later credits to the merchant first recover written-off amounts, crediting merchant losses.

Example: Birch has been paid out, so available is $0, unsettled $0 and reserve $96. A $400 refund posts Dr risk reserve $96, Dr due from merchant $304 / Cr PSP clearing $400, plus Dr FBO cash $304 / Cr operating cash $304. Birch's next $500 settlement pays $304 to due from merchant (with Dr operating cash / Cr FBO cash $304) and $196 to available, less holdback.

## Considered options

- **Let available go negative** (what ADR 0004 originally allowed). Rejected: the FBO identity would need a term for negative merchants, and the platform's exposure would be a sum over every merchant's sign.
- **Refuse refunds the merchant can't fund.** Rejected: ACH returns, and later chargebacks, can't be refused, so it adds a second behaviour without removing the first.
- **Draw the reserve before unsettled.** Rejected: it uses up the protection while the merchant's own money is still on its way.
- **Top up once a day at the FBO check.** Rejected: the ledger identity would be false during the day.
- **A standing platform buffer in the FBO account.** Rejected: it puts platform money in FBO, against ADR 0005.

## Consequences

- **ADR 0004 is amended:** merchant available carries `CHECK (balance >= 0)` like the other merchant accounts. The payout's guarded debit stays as the application-level check.
- **The bank-side shortfall is bounded by a day.** The ledger books the top-up immediately, but the real FBO balance can be short between the PSP netting a refund and that day's funding transfer. The design doc says so.
- **The FBO ledger identity holds once the PSP nets each refund.** In between, FBO cash exceeds merchant balances by refunds awaiting netting. This timing comes from the chart of accounts, not from this ADR. The reconciliation design should name it.
- **Simulated chargebacks post nothing.** They are training labels and an input to the merchant risk policy's chargeback rate. When the dispute-lifecycle stretch goal lands, a chargeback goes through this same waterfall, plus a chargeback fee.
- **No payout holds or tier moves here.** This ADR emits two merchant risk policy inputs, "platform covered a shortfall" and "debt written off". What the policy does with them belongs to [Merchant risk policy thresholds and tier moves](https://github.com/stvijayanand/plumb/issues/34) ([ADR 0002](0002-merchant-risk-rules-policy.md)).
- **New accounts:** due from merchant (per merchant, platform asset) and merchant losses (platform expense, per merchant).
