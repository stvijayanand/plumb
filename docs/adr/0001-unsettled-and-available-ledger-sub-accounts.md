---
status: accepted
---

# Unsettled and available are real ledger sub-accounts of merchant payable

Only settled funds may be paid out, so the ledger has to know which part of a merchant payable the PSP has actually delivered into the FBO account. We split each merchant's payable into two real ledger accounts, **unsettled** and **available**, and make every change between them a posted entry: capture credits unsettled, settlement moves funds from unsettled to available, reserve holdback moves them from available to the risk reserve and back on release, and a payout moves them from available to payouts in transit. Payouts draw only from available. We chose this over a derived "settled" flag because it makes payout eligibility a one-account balance lookup, gives every change in eligibility its own journal entry, and lets reconciliation check one number: total unsettled across merchants must equal what the PSP still owes the FBO account.

## Considered Options

- **One merchant payable account, with a settled flag on each posted entry.** The settlement job flips the flag when the PSP settlement file lands. Rejected: payout eligibility becomes a scan and sum over flagged entries, not a balance read; flipping a flag is an update to a ledger row, which breaks the append-only journal (see [#4](https://github.com/stvijayanand/plumb/issues/4)); and nothing in the journal records *when* funds became eligible or which settlement batch made them so.
- **One account, with unsettled and available as balance views** computed by joining entries to the PSP settlement file or to payment state. Rejected: the views encode settlement logic outside the ledger, so the zero-sum invariant no longer covers it. A payout lock must then guard a computed value rather than a balance row with a `CHECK` limit, and reconciliation would compare the settlement file to a view derived from that same file, which proves nothing.
- **Real sub-accounts (chosen).** Each state funds can be in is an account, and each transition is a balanced posted entry with its own idempotency key. Eligibility is the available balance. The unsettled total is an independent ledger figure that reconciliation compares to the PSP's own report.

## Consequences

- Settlement becomes a ledger posting, not just a reconciliation step: when a settlement file line matches a capture, the ledger posts unsettled → available for that merchant. An unmatched line is a reconciliation break, not a posting.
- There are more entries per payment (capture, settlement, holdback, release, payout), which adds to the hot-account load on the platform's clearing accounts that the load tests are meant to find.
- ACH needs nothing special: a succeeded debit posts to unsettled and moves to available when settlement lands, after the 2-day return window ([#11](https://github.com/stvijayanand/plumb/issues/11)).
- Out of scope here: which sub-account a refund or chargeback debits, and how far available may go negative, are for [#6](https://github.com/stvijayanand/plumb/issues/6). The full chart of accounts and worked postings are for [#3](https://github.com/stvijayanand/plumb/issues/3). Reserve percentages and timing are in [#10](https://github.com/stvijayanand/plumb/issues/10).
