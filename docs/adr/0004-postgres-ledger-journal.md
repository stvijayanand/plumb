---
status: accepted
---

# The ledger is a Postgres journal modelled on TigerBeetle, not TigerBeetle itself

Plumb needs a double-entry ledger with pending entries, idempotent postings and invariants the database enforces. TigerBeetle provides all three natively, but it would be a second datastore beside the Postgres that Plumb needs anyway for orchestration state, the outbox, risk and review data. We build the ledger as a Postgres journal that copies TigerBeetle's data model, because then a posting, the payment state change that caused it and its outbox row commit in one transaction. That is what makes "zero double posts under concurrent duplicate retries" provable in Phase 2, and it adds nothing to the single-instance stack of [ADR 0003](0003-single-instance-demo-deployment.md). TigerBeetle remains the named scale-out alternative ([findings](https://github.com/stvijayanand/plumb/blob/research/tigerbeetle-vs-postgres/docs/research/tigerbeetle-vs-postgres-ledger.md)).

## Considered Options

- **TigerBeetle for the journal, Postgres for everything else.** Rejected. Its two-phase transfers match Plumb's pending entries exactly, and it is Apache-2.0. But the recommended cluster is 6 replicas on separate machines with NVMe, which conflicts with ADR 0003. Docker needs `seccomp=unconfined`. Spring transactions don't extend to it, so every posting would become an outbox-plus-retry hop. Each multi-leg entry becomes a chain of linked transfers. And because balance limits are set per account, letting merchant available go negative for refunds but not for payouts would need the 5-transfer balance-bounds recipe.
- **Both, behind a ledger port.** Rejected: it doubles the set of invariants the property tests must prove, for a scale Plumb won't reach.
- **Postgres journal with TigerBeetle's model (chosen).** Postgres keeps the parts of TigerBeetle that matter here:
  - Pending entries are closed by separate post or void entries that name them. `UNIQUE (pending_entry_id)` lets a pending entry be closed only once.
  - Journal tables are append-only: the app role has no `UPDATE` or `DELETE`, and a trigger raises on any attempt.
  - A deferred constraint trigger checks at commit that each entry's lines sum to zero.
  - A pending entry carries `expires_at`, and a scheduled job writes the void entry when it lapses.

## Consequences

- **Entry ids are idempotency keys derived from the business event,** a hash of (payment id, ledger event), e.g. `capture` or `settlement:<line id>`, and unique. A replayed webhook or a retried capture maps to the same entry. The same id with a different payload is an error, as with TigerBeetle's `exists_with_different_*`. Hashed ids aren't time-sortable, so ordering comes from a separate commit sequence.
- **Each account has a balance row,** with pending and posted debit and credit totals, updated in the same transaction as the lines. The lines stay the source of truth. A property test and a nightly check assert that each balance row equals the sum of its lines. This stands in for the outline's "balances derived from entries with snapshots": the row is an always-current snapshot.
- **Concurrency is pessimistic, not the outline's optimistic versions.** A posting takes `SELECT … FOR UPDATE` on the balance rows it touches, in account-id order so two postings can't deadlock. Every capture touches PSP clearing and fee revenue. Under optimistic or serializable concurrency those rows would make most concurrent postings conflict and retry. Row locks turn the contention into plain lock queueing, which is easy to measure. Duplicate retries are rejected by the unique entry id, whatever the lock strategy.
- **Hot accounts are naive on purpose in Phase 2:** fee revenue and PSP clearing each get one balance row. The fix, either sharded sub-accounts or batched postings, is chosen from load-test data, and the before-and-after curve is a deliverable. Don't "fix" this early.
- **Balance limits can depend on the kind of entry, not only on the account.** Unsettled, risk reserve and payouts in transit carry `CHECK (balance >= 0)`. Merchant available has no `CHECK`, so a refund can push it negative. A payout debits available only under the row lock and only up to its current balance, so a payout can't overdraw it. What happens once available is negative is for [#6](https://github.com/stvijayanand/plumb/issues/6).
- **The database enforces the invariants only as a backstop.** Posting logic lives in the Java ledger module, the only code that writes entries, so jqwik property tests can exercise it. There are no stored procedures. Tests run against real Postgres via Testcontainers.
