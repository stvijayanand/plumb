# Plumb

Payment orchestration, double-entry ledger and real-time risk decisioning for a B2B billing platform that collects invoice payments from payers on behalf of merchants.

> **Status:** early design. The repo holds the domain model ([`GLOSSARY.md`](GLOSSARY.md)), the project outline (`Plumb-Outline.docx`) and agent tooling. No application code exists yet. Everything under Architecture, Tech stack and Roadmap is the **planned** design.

---

## What it does

- **Payment orchestration**: collects card and ACH payments from payers for merchants' invoices and moves the funds into the FBO account. Covers authorize, capture, refund, void and failed-payment retries.
- **Double-entry ledger**: the single source of truth for balances. Authorization holds are pending entries; every later money movement (capture, settlement, refunds, payouts, payout returns) is a posted entry.
- **Real-time risk decisioning**: a synchronous check in the authorization path returns approve, decline, review or step up. The merchant risk policy also sets each merchant's risk reserve percentage and places payout holds, based on refund rate, chargeback rate and account age.
- **Reconciliation and monitoring**: a daily three-way match against a simulated PSP settlement file, plus light compliance-style transaction monitoring.

Use the vocabulary in [`GLOSSARY.md`](GLOSSARY.md) in code, issues and docs, and avoid the synonyms it lists.

### Scope

| In scope | Out of scope |
| --- | --- |
| Card and ACH payments, refunds, voids, retries | Real card data (PSP test tokens and synthetic data only, to stay outside PCI scope) |
| Ledger as the source of truth for balances | A polished customer UI (a thin ops console for the review queue is enough) |
| Rules plus one ML model, with shadow mode and champion/challenger | Multi-currency and FX, real-time rails (RTP, FedNow), chargeback workflows (stretch goals) |
| Manual review queue whose outcomes become training labels | |
| Observability, SLOs, load tests and failure drills | |

---

## Architecture overview

The orchestrator owns every payment's state. Nothing moves until the risk service answers inside its latency budget. Everything downstream of the event stream is asynchronous, so review, reconciliation and training never slow the authorization path.

### Components

| # | Component | Language | Responsibility |
| --- | --- | --- | --- |
| 1 | Payment orchestrator | Java | Explicit state machine (`created → risk_checked → authorized → captured → settled`, with declined, in_review, voided, refunded and failed branches). Idempotency keys on every write API. Unknown-outcome handling for PSP timeouts. Transactional outbox to Kafka. Health-based failover across two simulated PSPs. |
| 2 | Double-entry ledger | Java | Append-only journal where every entry's lines sum to zero. Corrections are reversing entries. Pending vs posted balances. Invariants enforced in the database and checked by property-based tests. |
| 3 | Risk decision service | Python | Hard latency budget with per-rule fail-open or fail-closed policy. Rules engine plus one gradient-boosted model, combined into a decision with reason codes. Shadow mode. Every decision stored with inputs, model version and rule version so it can be replayed. |
| 4 | Review queue and ops console | Python / TS | Transactions scored "review" stay authorized but not captured. Analysts decide against an SLA timer, with auto-void on expiry. Outcomes feed back as training labels. |
| 5 | Reconciliation and monitoring | Python batch | Daily three-way match of ledger, orchestrator records and PSP settlement file. Breaks are classified and ticketed, not silently fixed. |
| 6 | PSP simulator | Java | Deterministic by default, with a chaos mode: latency, timeout after success, duplicate and out-of-order webhooks, partial settlements. |
| 7 | Payouts and merchant risk policy | Java | Payout lifecycle `scheduled → initiated → sent → paid`, with a `returned` branch that puts the money back in available. A simulated bank rail sends ACH-style transfers that can be paid, delayed, or returned days later (e.g. closed account); returns are the payout equivalent of a PSP timeout after success. The merchant risk policy is deterministic, not a model: thresholds on refund rate, chargeback rate and account age set each merchant's reserve percentage and trigger payout holds. It acts on the available balance, outside the auth path. Only settled funds are eligible for payout; the schedule (cadence, delay after capture, reserve percentage) is configuration, with values tracked in a ticket. |

### Design principles

- **Illegal state transitions are rejected**, not logged.
- **A PSP timeout is an "unknown" outcome**, resolved by status query or webhook, never by blind retry.
- **State changes and their events commit in one transaction** (transactional outbox), then publish.
- **Same feature definitions online and offline** to avoid training/serving skew.
- **Money is never edited**: pending entries are never changed and posted entries are only reversed.

---

## Tech stack

| Layer | Technology | Why |
| --- | --- | --- |
| Payments core, ledger and payouts | Java 21+, Spring Boot | Default at banks, card networks and FinTechs; virtual threads suit I/O-heavy PSP calls |
| Primary store | PostgreSQL | Strong transactions and constraints |
| Events | Apache Kafka (KRaft) | Outbox publishing, feature pipelines, audit stream |
| Risk service | Python, FastAPI | Where fraud and risk ML teams work |
| Model | LightGBM or XGBoost, MLflow registry | Standard for tabular fraud; versioned, replayable models |
| Online features | Redis | Low-latency rolling counters in the auth path |
| Offline data | Parquet, DuckDB | Cheap local analytics and training sets |
| Observability | OpenTelemetry, Prometheus, Grafana | Traces across the auth path, SLO dashboards |
| Testing | JUnit 5 + jqwik, Testcontainers, pytest, k6 | Invariants, real dependencies, load numbers |
| Deploy | Docker Compose locally; Terraform on AWS (one on-demand instance, destroyed after each demo) | Cloud supports the project without being the story; `demo-up` / `demo-down` scripts keep cost guarded |

---

## Planned project structure

```
plumb/
  docs/            # design doc, ADRs, runbooks, game-day reports, model card
  orchestrator/    # Java: payment API, state machine, PSP routing, outbox
  ledger/          # Java: journal, accounts, balances, invariants
  payouts/         # Java: scheduler, bank rail simulator, merchant policy
  psp-simulator/   # Java: deterministic + chaos modes
  psp-stripe/      # Java: Stripe test-mode adapter (phase 5)
  risk/            # Python: decision API, rules, scoring, shadow mode
  features/        # Python: online and offline feature definitions
  ml/              # Python: training, evaluation, cost curves, MLflow
  recon/           # Python: settlement matching, monitoring rules
  console/         # thin review-queue UI
  infra/           # Compose, Terraform, Grafana dashboards
  loadtest/        # k6 scenarios
```

None of these directories exist yet. See [Repository layout](#repository-layout) for what is in the repo today.

---

## Domain language

[`GLOSSARY.md`](GLOSSARY.md) defines the terms, grouped as:

| Group | Terms |
| --- | --- |
| Parties | Platform, Merchant, Payer |
| Money held | FBO account, Merchant payable, Risk reserve, Payout, Bank rail, Payouts in transit, Payout return, Unsettled balance, Available balance, Payout hold |
| Ledger | Pending entry, Posted entry |
| Risk | Merchant risk policy |

---

## Metrics and SLOs

Starting targets, to be tuned once load-test data exists.

| Area | Metric | Target |
| --- | --- | --- |
| Correctness | Ledger entries that don't sum to zero | 0, enforced |
| Correctness | Duplicate charges under concurrent retries | 0 |
| Latency | Risk decision, p99 | < 50 ms |
| Latency | Payment API, p99, excluding PSP time | < 250 ms |
| Availability | Auth path success when risk service is down | Per documented fail-open/closed policy |
| Reconciliation | Unexplained breaks after daily run | 0 open past 24 hours |
| Risk | Fraud caught; false-positive and approval rate | Report by count and by dollars |
| Risk | Share of transactions sent to review | Capped (e.g. 2%) to fit analyst capacity |

The headline risk result is a **cost curve**, not an accuracy score: total business cost at each score threshold (missed fraud including chargeback fees, false declines at lost margin, and review cost), with the chosen threshold marked and justified.

---

## Planned ADRs

Recorded in `docs/adr/` as each decision is made.

| Decision | Alternatives |
| --- | --- |
| Ledger store | PostgreSQL vs a purpose-built ledger database such as TigerBeetle |
| Risk check placement | Synchronous vs async with a post-auth hold |
| Failure policy | Fail-open vs fail-closed, per rule |
| Event publishing | Kafka with outbox vs direct publishing |
| Launch scoring | Rules only vs rules plus model |
| PSP abstraction | One interface: simulator first, Stripe test-mode adapter later |

---

## Roadmap

Five phases over 26 weeks. A phase starts only when the previous phase's gate is met. Phase 2 (ledger and idempotency) is the longest on purpose.

| Phase | Weeks | Build | Gate | Status |
| --- | --- | --- | --- | --- |
| 1. Foundations | 1–2 | Design doc v1, domain model, chart of accounts; monorepo, CI, Docker Compose stack; PSP simulator skeleton | Design doc reviewed by a peer or mentor | 🔄 In progress (domain model done) |
| 2. Ledger + core | 3–8 | Double-entry ledger with database-enforced invariants; payment API, state machine, idempotency keys; transactional outbox to Kafka; PSP routing with health-based failover | Zero double postings under concurrent duplicate retries; property tests green | ⬜ |
| 3. Risk decisions v1 | 9–13 | Risk API in the auth path with a latency budget; rules engine and Redis online features; decision log with reason codes and replay | p99 within budget under load; any past decision is replayable; fail-open/fail-closed ADR written | ⬜ |
| 4. Model + review | 14–19 | Model trained on public fraud data, tracked in MLflow; shadow mode with champion vs challenger; review queue with capture holds and label feedback | Cost curve and chosen threshold published; shadow-vs-live report written | ⬜ |
| 5. Harden + publish | 20–26 | Payouts, bank rail simulator, merchant risk policy; reconciliation, monitoring rules, Stripe test-mode adapter; chaos drills, SLO dashboards, load tests; `demo-up` / `demo-down` scripts with cost guardrails; write-ups, demo video | Two game-day reports, two public posts, resume and LinkedIn live | ⬜ |

Planned documentation artifacts: design doc (RFC), ADRs, failure-mode table, two game-day reports, load-test report, model card and a decision-replay demo.

Stretch goals, in rough order: chargeback and dispute lifecycle, multi-currency ledger with FX, real-time rails, graph features for linked-account fraud rings.

---

## Repository layout

| Path | Contents |
| --- | --- |
| `GLOSSARY.md` | The domain language: parties, money held, ledger and risk terms |
| `Plumb-Outline.docx` | Project outline: scope, architecture, stack, build plan, metrics |
| `docs/adr/` | Architecture decision records (created as decisions are made) |
| `docs/agents/` | How coding agents use the issue tracker, triage labels and domain docs |
| `.claude/skills/`, `.agents/skills/` | Agent skills used to work on this repo |
| `skills-lock.json` | Pinned versions of the installed skills |
| `CLAUDE.md` | Entry point for coding agents |

---

## Contributing

Issues are tracked in [GitHub Issues](https://github.com/stvijayanand/plumb/issues). New issues start as `needs-triage`; the full label set is described in [`docs/agents/triage-labels.md`](docs/agents/triage-labels.md).

Before working in an area, read `GLOSSARY.md` and any ADRs in `docs/adr/` that touch it.

Keep this README current as the project moves through its phases. When a change alters what Plumb does, how the repo is laid out, or how to set up, build, run or test it, update the matching section here (including the **Status** note) in the same change.
