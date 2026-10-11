# Plumb

Payment orchestration, double-entry ledger and real-time risk decisioning for a B2B billing platform that collects invoice payments from payers on behalf of merchants.

> **Status:** early design. The repo holds the domain model and agent tooling. No application code exists yet.

## What Plumb does

- **Payment orchestration**: collects payments from payers for merchants' invoices and moves the funds into the FBO account.
- **Double-entry ledger**: records authorization holds as pending entries and every later money movement (capture, settlement, refunds, payouts, payout returns) as posted entries.
- **Risk decisioning**: the merchant risk policy sets each merchant's risk reserve percentage and places payout holds, based on refund rate, chargeback rate and account age.

See [`GLOSSARY.md`](GLOSSARY.md) for the definitions of these terms. Use the glossary's vocabulary in code, issues and docs, and avoid the synonyms it lists.

## Repository layout

| Path | Contents |
| --- | --- |
| `GLOSSARY.md` | The domain language: parties, money held, ledger and risk terms |
| `docs/adr/` | Architecture decision records (created as decisions are made) |
| `docs/agents/` | How coding agents use the issue tracker, triage labels and domain docs |
| `.claude/skills/`, `.agents/skills/` | Agent skills used to work on this repo |
| `skills-lock.json` | Pinned versions of the installed skills |
| `CLAUDE.md` | Entry point for coding agents |

## Contributing

Issues are tracked in [GitHub Issues](https://github.com/stvijayanand/plumb/issues). New issues start as `needs-triage`; the full label set is described in [`docs/agents/triage-labels.md`](docs/agents/triage-labels.md).

Before working in an area, read `GLOSSARY.md` and any ADRs in `docs/adr/` that touch it.
