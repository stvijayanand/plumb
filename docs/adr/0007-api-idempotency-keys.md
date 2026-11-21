---
status: accepted
---

# API idempotency keys live in Postgres, conflict with 409 and replay the original response

Every mutating Plumb API call carries an **idempotency key**, so a client that retries after a timeout gets the result of its first request instead of a second charge. We store the keys in a Postgres table in the orchestrator's database, written in the same transaction as the payment, so "zero double posts under concurrent duplicate retries" (the Phase 2 gate) rests on a unique constraint and a commit, not on a cache. A duplicate that arrives while the first request is still running gets **409** at once rather than waiting. A replay returns the **original** saved response, not the payment's current state, so a client retry never triggers a PSP call.

## Three layers of idempotency

| Layer | Key | Happens once per | Defined in |
| --- | --- | --- | --- |
| API | `Idempotency-Key` header, scoped to the merchant | client intent (pay this invoice, refund this amount) | this ADR |
| PSP | per-attempt PSP idempotency key | PSP call within an attempt | [payment state machine](https://github.com/stvijayanand/plumb/issues/12) |
| Ledger | entry id derived from the business event | ledger event | [ADR 0004](0004-postgres-ledger-journal.md) |

A payment keeps its API key across all its attempts; each attempt gets its own PSP key. Webhooks and other PSP facts skip the API layer: the state machine applies, ignores or raises them as reconciliation breaks.

## The rules

1. **Required.** Every mutating endpoint (create payment, capture, void, cancel, refund, analyst approve/decline) needs `Idempotency-Key`; otherwise 400. GETs ignore it. Keys are opaque strings of up to 255 characters (UUIDv4 recommended). Autopay derives its keys, e.g. `autopay:<invoice id>:<cycle>`, so a scheduler rerun can't charge twice.
2. **Scope.** Unique on `(merchant_id, key)`. The endpoint is stored with the record but isn't part of the key.
3. **Record.** `merchant_id, key, endpoint, request_hash` (SHA-256 over the method, path and canonicalised JSON body), `status` (in progress / completed), `locked_until, recovery_point, resource_id, response_code, response_body, created_at`.
4. **Mismatch.** Same key with a different hash or endpoint: **422 `idempotency_key_reused`**. Nothing runs and nothing is stored.
5. **Concurrency.** The first request claims the key with `INSERT … ON CONFLICT DO NOTHING`, committed before any work starts, and sets `locked_until = now + 30 s`. A duplicate arriving while that lock is live gets **409 `request_in_progress`** with `Retry-After: 1`.
6. **What is saved.** A response is saved only once the endpoint has started executing. A 400 validation failure saves nothing, so the client can fix the body and reuse the key. Every later outcome is saved and replayed: 2xx, business 4xx, and a payment created as *declined*.
7. **Replay.** Returns the saved response unchanged. If the first response showed *authorization unknown*, the replay shows it too; clients read current state with `GET`. Unknown outcomes resolve only on the server.
8. **Retention.** Records are purged after 7 days. A payment also keeps a permanent `UNIQUE(merchant_id, idempotency_key)`, so replaying an expired create-payment key returns **409 `idempotency_key_expired`** with the existing payment id and never creates a second payment.

## Crash recovery

The handler commits progress in steps. Step one claims the key and inserts the payment in *created* in one transaction, and each later step commits the payment's state with a new `recovery_point`. If a request dies with its record still *in progress*, a replay after `locked_until` takes over the lock (`UPDATE … WHERE locked_until < now()`) and resumes from the payment's state:

- *created*: reuse the attempt's logged risk decision if there is one, otherwise score it.
- *authorizing*, *capturing*, *submitting* or any *_unknown* state: resend with the **same** per-attempt PSP key, never start a new attempt.
- any later state: build the response from the current payment and save it.

The lock must outlast the risk budget plus the PSP client timeout (10 s) plus a margin. If a live request outlives it anyway, the PSP key and the ledger entry ids still prevent a double charge or double post. The lock only prevents duplicated work.

Idempotency records are not swept: they finish on replay or expire. Payment progress must not depend on the client coming back, so a separate sweeper resolves payments stuck in an in-flight or unknown state longer than the lock, by status query or by resending with the same PSP key. It belongs to the reconciliation design.

## Considered options

- **Redis for the key store.** Rejected: it can't commit atomically with the payment row, and it can lose acknowledged writes. Atomicity is what makes the gate provable.
- **Block the duplicate until the first request finishes.** Rejected: a request can last as long as a PSP timeout, and every blocked duplicate holds a thread and a connection. 409 is what Stripe and the IETF Idempotency-Key draft do, and it's easy to test under concurrency.
- **Replay current state.** Rejected: replays would stop being deterministic, and "replay means resume" tempts the API layer to drive PSP calls that belong to the state machine.
- **Optional keys (Stripe's default).** Rejected: retries without a key are exactly the double charges the gate forbids.
- **24-hour retention, like Stripe.** Rejected as too short for weekend retries and autopay reruns. The permanent unique constraint on payments covers the case where an expired key is reused.
