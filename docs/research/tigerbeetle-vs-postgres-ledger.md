# TigerBeetle vs a Postgres journal for Plumb's ledger

Research for [#4](https://github.com/stvijayanand/plumb/issues/4), part of map [#2](https://github.com/stvijayanand/plumb/issues/2). Sources were read on 2026-10-10 against TigerBeetle 0.17.9 (latest release, 2026-07-06) and PostgreSQL 18 docs. Vocabulary follows `GLOSSARY.md`.

## Short answer

- TigerBeetle's two-phase transfers match Plumb's **pending entry** model almost one-to-one. A pending transfer is immutable. A separate post or void transfer that names it via `pending_id` closes it. Partial posts release the remainder, `timeout` gives expiry, and a pending transfer can be resolved only once.
- TigerBeetle enforces balance limits **per account**, not per transfer, through `debits_must_not_exceed_credits` or `credits_must_not_exceed_debits`. A merchant's available balance must be allowed to go negative (a refund after a payout) while a payout must not overdraw it, so that account cannot simply carry a flag. Plumb would need TigerBeetle's linked "balance bounds" recipe or a different account split.
- Idempotency is built in. The 128-bit transfer `id` is the idempotency key, and a retry returns `exists`, or `exists_with_different_*` if the payload changed.
- The Java client works with Java 21. It is thread-safe and batches requests. Production runs on Linux only.
- **Operational cost is the main drawback.** The recommended production cluster is 6 replicas, each on its own machine with ECC RAM (6 GiB minimum) and local NVMe. AWS has no managed TigerBeetle; TigerBeetle's own managed service is sold to enterprises. Plumb also needs Postgres anyway for orchestration state, risk and metadata, and TigerBeetle's own architecture docs assume a Postgres control plane next to it.
- The license is Apache-2.0.
- Postgres can enforce the same invariants with the measures in the [table](#what-postgres-needs-to-match-tigerbeetle) below: append-only tables, a deferred constraint trigger for "entry lines sum to zero", a balance row per account with `CHECK (balance >= 0)` where the account requires it, row locks (or an optimistic `version` column, or `SERIALIZABLE` with retries on 40001), a `UNIQUE` idempotency key, and `UNIQUE(pending_entry_id)` on closing entries.
- **Recommendation for Plumb:** build a Postgres journal that copies TigerBeetle's data model (pending, post and void entries linked by `pending_id`; client-generated 128-bit ids as idempotency keys; per-account balance-limit flags; append-only lines). Mention TigerBeetle in the design doc as the scale-out alternative.

## 1. Two-phase transfers vs Plumb pending entries

What TigerBeetle does ([two-phase transfers](https://docs.tigerbeetle.com/coding/two-phase-transfers/), [Transfer reference](https://docs.tigerbeetle.com/reference/transfer/)):

- A transfer with `flags.pending` reserves its amount in the accounts' `debits_pending` and `credits_pending`. It leaves `debits_posted` and `credits_posted` unchanged.
- Resolving a pending transfer always creates a **new** transfer with its own `id` and `pending_id = <pending id>`, flagged `post_pending_transfer` or `void_pending_transfer`. Transfers cannot be modified or deleted. Corrections are made with new reversing transfers.
- **Posting less than the pending amount posts only that amount, and the remainder returns to the original accounts.** Posting more than the pending amount fails with `exceeds_pending_transfer_amount`. Passing `AMOUNT_MAX` posts the full amount.
- Void releases the full reservation. The optional `timeout` (seconds, relative to the pending transfer's timestamp) expires the transfer. After that it can be neither posted nor voided, and removing the reservation is "best-effort, not exact" in timing.
- A pending transfer can be resolved only once. A second attempt returns `pending_transfer_already_posted`, `pending_transfer_already_voided` or `pending_transfer_expired`.
- The post or void transfer must match the pending transfer's accounts, ledger and code (zero fields are filled in from the pending transfer).

How this maps to Plumb's settled rules (map #2, "Entry status vs balance"):

| Plumb rule | TigerBeetle mechanism | Fit |
|---|---|---|
| Authorization writes a pending entry, never edited | `pending` transfer; transfers are immutable | Exact |
| Capture closes it and writes a posted entry | `post_pending_transfer` with `pending_id` | Exact |
| Void or expiry closes it with nothing posted | `void_pending_transfer`, or `timeout` | Exact. Expiry is best-effort in timing, and Plumb's simulated PSP decides the real auth expiry anyway. |
| Partial capture lets the remainder lapse | Post with a smaller amount; the remainder is released | Exact |
| A pending entry counts toward no balance a merchant can be paid from | Pending amounts sit in `*_pending` fields. The app derives balances from `*_posted` fields. | Good, with one caveat: pending **debits** count toward the debit account's limit check (see §2). A pending credit to merchant unsettled does not affect any payout limit. |

A structural difference: **a TigerBeetle transfer has exactly one debit account and one credit account.** A multi-leg Plumb entry, such as capture crediting merchant unsettled and platform fee revenue, becomes several transfers joined by the `linked` flag, which makes them all-or-nothing ([Transfer flags](https://docs.tigerbeetle.com/reference/transfer/), [create_transfers](https://docs.tigerbeetle.com/reference/requests/create_transfers/)). In TigerBeetle the posted fee leg cannot be part of the pending/post pair unless the authorization also reserves the fee leg as its own pending transfer.

## 2. Balance constraints and negative balances

- Account flags ([Account reference](https://docs.tigerbeetle.com/reference/account/)): `debits_must_not_exceed_credits` rejects a transfer if `debits_pending + debits_posted + amount > credits_posted`. `credits_must_not_exceed_debits` is the mirror rule. The two flags are mutually exclusive. With no flag, the balance may go negative. Failures return `exceeds_credits` or `exceeds_debits`.
- **Pending amounts count against the limit**, so reservations reduce headroom. Because the check runs when the pending transfer is created, posting or voiding can never break the limit ([two-phase transfers](https://docs.tigerbeetle.com/coding/two-phase-transfers/)).
- `balancing_debit` and `balancing_credit` transfer "as much as possible up to `amount`" within the limit.
- The [balance bounds recipe](https://docs.tigerbeetle.com/coding/recipes/balance-bounds/) enforces arbitrary bounds with a chain of 5 linked transfers: the real transfer, then a pending `balancing_*` probe against a control account, then voiding the probe, with funding and reset legs around it.
- The `closed` flag (set through a `closing_debit` or `closing_credit` pending transfer) blocks further transfers except voids.

What this means for Plumb's merchant payable:

- **Merchant available** must allow a negative balance, because a refund can arrive after a payout. A payout, however, must not draw it below zero. Since the flag is per account, setting `debits_must_not_exceed_credits` on available would also reject the refund. Options:
  1. Leave available unflagged and check payouts with the linked balance-bounds chain (or a `balancing_debit` payout).
  2. Leave it unflagged and check in the application while serialising payouts per merchant.
  3. Flag available, and post refunds that exceed it to a separate "due from merchant" account.

  This is a chart-of-accounts decision for the ledger grilling ticket, not for this research.
- **Merchant unsettled** can safely carry `debits_must_not_exceed_credits`, because settlement must never move more than was captured.
- Amounts are unsigned 128-bit integers in minor units (asset scale 2 for USD) ([data modeling](https://docs.tigerbeetle.com/coding/data-modeling/)). This fits Plumb's single currency.

## 3. Java client

From the [Java client docs](https://docs.tigerbeetle.com/coding/clients/java/):

- The Maven artifact is `com.tigerbeetle:tigerbeetle-java`. It requires Java 11 or later, so Plumb's Java 21 is fine.
- **Production runs on Linux 5.6 or later only.** macOS and Windows are supported for development.
- It uses batch objects (`AccountBatch`, `TransferBatch`, `IdBatch`). The server batch limit is 8189 events. One client instance is thread-safe and should be shared. It automatically batches requests from concurrent threads.
- 128-bit ids accept `long` pairs, `byte[]`, or `UUID` through `UInt128`. Amounts and balances are `BigInteger`.
- There is no JDBC or Spring Data integration. Spring transactions do not extend to TigerBeetle, so a Postgres outbox plus idempotent transfer ids is the consistency pattern.

## 4. Idempotency

- The `id` field is unique across the cluster, and "at most one Transfer exists per `id`" ([Transfer reference](https://docs.tigerbeetle.com/reference/transfer/)).
- A retry with the same id returns `exists`. If the same id arrives with a different field, the result is one of `exists_with_different_{flags,pending_id,timeout,debit_account_id,credit_account_id,amount,user_data_*,ledger,code}`. If the original attempt failed with a transient error, the result is `id_already_failed` ([create_transfers](https://docs.tigerbeetle.com/reference/requests/create_transfers/)).
- Recommended practice: the **client** generates the id and persists it before submitting, then retries with the same id ([reliable transaction submission](https://docs.tigerbeetle.com/coding/reliable-transaction-submission/)). Ids should be time-sortable: a 48-bit millisecond timestamp plus 80 random bits ([data modeling](https://docs.tigerbeetle.com/coding/data-modeling/)).
- For Plumb, transfer ids could be derived deterministically from (payment id, ledger event), for example a UUIDv7 or a hash, so a replayed PSP webhook maps to the same id.

## 5. Operational cost

**Local development:**

- TigerBeetle is a single static binary, with builds for Linux x86_64 and aarch64, macOS universal, and Windows x86_64 ([installing](https://docs.tigerbeetle.com/operating/installing/)).
- Quick start: `tigerbeetle format --cluster=0 --replica=0 --replica-count=1 --development ./0_0.tigerbeetle`, then `tigerbeetle start --addresses=3000 --development ...` ([quick start](https://docs.tigerbeetle.com/start/)).
- The Docker image is `ghcr.io/tigerbeetle/tigerbeetle`. It needs `--security-opt seccomp=unconfined`, because Docker 25+ blocks io_uring, and it may need the `IPC_LOCK` capability or a memlock ulimit. The docs advise running the binary directly rather than in Docker ([Docker](https://docs.tigerbeetle.com/operating/deploying/docker/)). This affects Plumb's docker-compose and Testcontainers setup.

**Production and AWS:**

- The recommended cluster is **6 replicas**, ideally spread over 3 sites with 2 replicas each. Each replica's data file needs its own disk and its own machine ([cluster](https://docs.tigerbeetle.com/operating/cluster/)).
- Each replica needs 1 core plus a spare core for the OS, **ECC RAM with at least 6 GiB** (16–32 GiB recommended), local NVMe strongly recommended (EBS works but is slower), and at least 1 Gbps networking ([hardware](https://docs.tigerbeetle.com/operating/hardware/)).
- On AWS that means 6 EC2 instances with NVMe instance store, such as the I4i, I4g or I3 families ([EC2 storage optimized](https://docs.aws.amazon.com/ec2/latest/instancetypes/so.html)), or EBS-backed instances. Plumb would have to run and upgrade them itself.
- There is no AWS-managed offering. TigerBeetle's [managed service](https://docs.tigerbeetle.com/operating/deploying/managed-service/) is enterprise-only, and you contact sales to get it.
- A 1-replica cluster runs (the quick start uses one) but has no fault tolerance.
- **Postgres** is needed by Plumb regardless: orchestration state machines, the outbox, risk and review data, and metadata. TigerBeetle's [system architecture](https://docs.tigerbeetle.com/coding/system-architecture/) itself places Postgres beside TigerBeetle as the control plane. A Postgres ledger adds no new infrastructure: it is one RDS instance (or a container locally) that Plumb already pays for.

## 6. Licensing

TigerBeetle is **Apache-2.0** (GitHub repo license, `tigerbeetle/tigerbeetle`). The latest release is 0.17.9 (2026-07-06), still on a 0.x version line. PostgreSQL ships under the PostgreSQL License, a permissive license. Licensing blocks neither option.

## What Postgres needs to match TigerBeetle

| TigerBeetle invariant | Postgres enforcement |
|---|---|
| Transfers are immutable | Use append-only `journal_entry` and `entry_line` tables. `REVOKE UPDATE, DELETE` from the app role, and add a `BEFORE UPDATE OR DELETE` trigger that raises. Corrections are new reversing entries. |
| Every entry balances (debits = credits) | A cross-row rule cannot be a `CHECK`. Postgres "does not support CHECK constraints that reference table data other than the new or updated row" ([constraints](https://www.postgresql.org/docs/current/ddl-constraints.html)). Use a **constraint trigger** `AFTER INSERT ... DEFERRABLE INITIALLY DEFERRED FOR EACH ROW`, which fires at the end of the transaction ([CREATE TRIGGER](https://www.postgresql.org/docs/current/sql-createtrigger.html)), to check that the lines of the entry sum to zero. Alternatively, allow inserts only through a stored function that validates the whole entry. |
| Per-account balance limits (`debits_must_not_exceed_credits`) | Keep an `account_balance` row per account with posted and pending debit and credit columns plus a per-account limit flag, and update it in the same transaction as the lines. A same-row `CHECK` is allowed, e.g. `CHECK (NOT debits_must_not_exceed_credits OR debits_pending + debits_posted <= credits_posted)`. Accounts that may go negative, like merchant available, leave the flag off. |
| Atomic, race-free balance updates | Choose one of three:<br>(a) Read Committed (the default) with `SELECT ... FOR UPDATE` on the balance rows, locked in a fixed order (e.g. by account id) to avoid deadlocks.<br>(b) An optimistic `version` column: `UPDATE ... SET ..., version = version + 1 WHERE id = ? AND version = ?`, retrying when 0 rows update.<br>(c) `SERIALIZABLE`, retrying the whole transaction on SQLSTATE `40001` ([transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html)).<br>Under Read Committed a plain `UPDATE ... WHERE` re-checks its condition on the newly committed row version, so `UPDATE ... SET balance = balance - :x WHERE id = :a AND balance >= :x` is a safe single-row conditional debit. |
| Idempotent transfer id | Add `UNIQUE` on `entry.id` or on an idempotency key, and use `INSERT ... ON CONFLICT DO NOTHING RETURNING id` (it returns no row on a conflict) ([INSERT](https://www.postgresql.org/docs/current/sql-insert.html)). On a conflict, re-read the stored entry and compare payloads to mimic `exists_with_different_*`. |
| A pending transfer resolves once, for at most its amount | Give the closing entry a `pending_entry_id` column with `UNIQUE (pending_entry_id)`, so it can be posted or voided only once. Check amount ≤ pending amount, and same accounts, in the inserting function while holding a `FOR UPDATE` lock on the pending row. |
| Pending reservations count against limits | Add and remove pending amounts on `account_balance` in the same transaction as the pending entry and its closing entry. |
| Expiry (`timeout`) | Store `expires_at` on the pending entry. A scheduled job writes a void (closing) entry for expired holds. Reads can also treat expired holds as lapsed. |
| Linked (all-or-nothing) multi-transfer | This comes free: a Postgres transaction is atomic, and one entry can have many lines. |

## Open points for the ledger grilling ticket

- Pick the chart-of-accounts treatment for a negative merchant available balance against "payouts never overdraw" (§2, options 1–3).
- Decide which accounts carry the pending authorization leg: for example a PSP clearing account debited and merchant unsettled credited, or the fee split as well.
- Pick the concurrency strategy for hot accounts, such as the platform fee revenue account. Row locks serialise on that row. Hot-account techniques (sharded sub-accounts, or deriving balances from lines) may matter in load tests.
