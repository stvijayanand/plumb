---
status: accepted
---

# Merchant risk is a rules policy, not a model

Plumb decides merchant risk with the **merchant risk policy**: deterministic rules that read a merchant's refund rate, chargeback rate and account age and act on the money the platform already holds for that merchant. The policy moves the merchant between merchant tiers (new / established), raises its risk reserve percentage, or places a payout hold on its available balance. It runs asynchronously in the payouts component, outside the auth path, and never approves or declines a payer's payment. Payer transactions are scored separately by the risk decision service (rules plus one model). We chose rules because the merchant side of Plumb has no labelled data, few decisions and a strong need for explanations, and because the outline's signal for ML is already met by the payer model.

## Considered options

- **Merchant risk model for bust-out detection** (a model that scores merchants for patterns such as a sudden jump in volume or ticket size just before refunds and chargebacks spike). Deferred as a stretch goal, not rejected:
  - **No training data.** Public fraud datasets (IEEE-CIS, PaySim, ULB) label payer transactions, not merchant bust-outs. A merchant model would train on synthetic merchants built from the same rules it is meant to beat, which proves nothing.
  - **Too few events to evaluate.** A demo has tens of merchants and a handful of bust-outs, so precision and recall would be noise. The payer model has the volume to support a cost curve and shadow mode; a merchant model does not.
  - **Actions must be explainable.** Holding a merchant's payouts withholds money the platform owes it. Each hold or reserve increase needs a reason a merchant and an auditor can read ("chargeback rate 1.2% over 30 days, above threshold"), which threshold rules give directly.
  - **Scope.** A second model means a second feature pipeline, training job, model card and shadow comparison. The outline caps the project at one model.
- **Merchant risk in the auth path** (decline payer payments to risky merchants). Rejected: it adds latency to every payment for a signal that changes over days, and it punishes payers for merchant behaviour. Holding funds after capture limits the platform's loss without that cost.

## Consequences

- **Settings only the policy changes.** A merchant's tier and its payout holds (placing and releasing) change only through a recorded policy decision with its reason. The per-merchant override from [Payout schedule and reserve configuration](https://github.com/stvijayanand/plumb/issues/10) cannot change either.
- **The reserve has a floor, not an owner.** The per-merchant override may set a reserve, for example one agreed in a merchant's contract, recorded with who set it and why. The policy's reserve is the tier default unless a recorded policy decision has raised or lowered it, and the effective reserve is the higher of the policy's reserve and the override. An override can tighten a reserve but never loosen one; only a policy decision can take a merchant below its tier default.
- **Thresholds are configuration**, kept in a table like the tier settings, and replayable: given a merchant's history and the threshold table, the policy's decision can be reproduced. The threshold values, demotion and hold release are owned by [Merchant risk policy thresholds and tier moves](https://github.com/stvijayanand/plumb/issues/34), not this ADR.
- **Bust-out exposure is bounded by the reserve and payout delay, not detected.** A merchant that ramps up and disappears inside the new-tier window is covered only by what the reserve and the T+7 delay still hold. The design doc should say so.
- **A merchant model can be added later without rework**: it would feed a score into the same policy as one more input, run first in shadow mode, and the actions (tier, reserve, hold) stay where they are.
