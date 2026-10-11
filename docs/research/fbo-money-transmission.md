# FBO accounts and money transmission framing

Research for ticket #7 (map #2). Question: how do real B2B billing platforms that collect on behalf of merchants hold and pay out funds (FBO accounts, PSP platform products such as Stripe Connect, payfac and sponsor-bank arrangements)? What are the US money transmission implications of each, at a framing level? Which arrangement should Plumb model so the design doc reads credibly?

> **Framing only, not legal analysis.** This note summarizes what primary sources say so the design doc can use accurate vocabulary. It is not legal advice and makes no claim about whether any real business needs a license. Money transmission is decided state by state, on facts and circumstances, and the federal rulings cited bind only the party that requested them.

Vocabulary follows `GLOSSARY.md`: *platform*, *merchant*, *payer*, *FBO account*, *payout*, *risk reserve*, *merchant payable*.

## Answer in brief

- **Three real arrangements.** (1) The platform uses a PSP's platform product (Stripe Connect), and the PSP, a licensed money transmitter, holds the funds. (2) The platform holds funds itself in an FBO account at a bank and pays merchants out over ACH, relying on money transmission exemptions. (3) The platform registers as a card-network payment facilitator under a sponsoring acquirer and receives settlement for its sponsored merchants. Sponsor banks sit under (2) and (3) as the institution that holds the FBO account or sponsors network and ACH access.
- **Federal framing.** "Money transmission services" means accepting funds from one person *and* transmitting them to another person or location ([FIN-2014-R009](https://www.fincen.gov/sites/default/files/administrative_ruling/FIN-2014-R009.pdf), citing 31 CFR 1010.100(ff)(5)). A payment processor escapes the definition only if all four conditions of the **payment processor exemption** hold. The one that most often fails is that funds must move *in and out* through clearance and settlement systems that admit only BSA-regulated financial institutions (for example ACH).
- **State framing.** The CSBS Model Money Transmission Modernization Act (MTMA), now enacted in whole or in part by 31 states ([CSBS](https://www.csbs.org/csbs-money-transmission-modernization-act-mtma)), defines money transmission to include "receiving money for transmission". It exempts an **agent of a payee** that collects for goods or services under a written agreement, provided the payer's obligation is extinguished when the agent receives the money ([MTMA §3.01(b)](https://www.csbs.org/sites/default/files/2023-02/CSBS%20Money%20Transmission%20Modernization%20Act.pdf)).
- **Recommendation for Plumb.** Model arrangement (2). A PSP (simulated, Stripe-like) acquires card payments and settles net into the platform's FBO account at a sponsor bank. The platform acts as the merchants' agent of payee and pays out by ACH only. Name the Stripe Connect alternative in the doc as the "buy" option. This matches what map #2 already settled: platform-held funds, an FBO account, ACH-style payouts with returns, and payouts in transit.

## 1. The arrangements

### 1a. PSP platform product (Stripe Connect)

The PSP onboards each merchant as a *connected account*. The PSP keeps the balances and runs the payouts. The platform gets an API over money it never holds directly.

- **Charge types decide whose balance the money lands in.** With direct charges, "the payment appears in the connected account's balance, not in your platform's balance", and refunds and chargebacks reduce the connected account's balance. With destination charges, and with separate charges and transfers, the charge lands in the platform's Stripe balance and a transfer moves the merchant's share. "Refunds and chargebacks reduce your platform's balance" ([Stripe: charges](https://docs.stripe.com/connect/charges)).
- Stripe names *"a SaaS platform, such as an accounting platform that enables invoice payments"* as the typical direct-charges case. That is Plumb's shape ([Stripe: charges](https://docs.stripe.com/connect/charges)).
- **The platform's account configuration sets its liability.** "Your choice of account type determines your platform's liability for fraud, abuse, negative balances, and identity verification." On legacy Express and Custom accounts the platform carries fraud and dispute liability ([Stripe: account types](https://docs.stripe.com/connect/accounts)).
- **Payouts.** Charges build up in the connected account's balance and pay out on a daily rolling basis, or on a schedule or manually as the platform sets. A Payout moves through `pending → in_transit → paid`, with `failed` and `canceled` branches ([Stripe: payouts to connected accounts](https://docs.stripe.com/connect/payouts-connected-accounts)).
- **Who is the money transmitter.** Stripe's US stored-balance products are provided by *Stripe Payments Company* (NMLS #1280479), which Stripe's terms describe as a state-licensed money transmitter and federally registered MSB, with funds held at partner banks ([Stripe Payments Company licenses](https://stripe.com/legal/spc/licenses), [SPC terms](https://stripe.com/legal/spc)). The licensed party holds the funds, so the platform does not.

### 1b. Platform-held FBO account at a (sponsor) bank

The platform opens a custodial account at an FDIC-insured bank, titled "[Platform] FBO its merchants". It keeps its own sub-ledger of who owns each dollar and instructs payouts over ACH. The bank is often called the *sponsor bank* because it also gives the platform ACH origination access.

- **Pass-through insurance depends on records.** FDIC pass-through coverage generally requires three things. The bank's records must disclose the custodial nature of the account. The owners and their interests must be ascertainable from bank or agent records. The agency relationship must be genuine ([FDIC: pass-through coverage](https://www.fdic.gov/financial-institution-employees-guide-deposit-insurance/pass-through-deposit-insurance-coverage); 12 CFR 330.5, 330.7).
- **Regulators are tightening this after Synapse.** In September 2024 the FDIC proposed a rule for custodial accounts with transactional features. Banks would have to maintain records that identify the owner of each dollar, "including a requirement to reconcile the account for each individual owner on a daily basis." The rule would apply even when a third party keeps those records. The FDIC Chairman framed the purpose as making sure "banks know the actual owner of deposits placed in a bank by a third party such as Synapse" ([FDIC press release](https://www.fdic.gov/news/press-releases/2024/fdic-proposes-deposit-insurance-recordkeeping-rule-banks-third-party); [89 FR 80135](https://www.federalregister.gov/documents/2024/10/02/2024-22565/recordkeeping-for-custodial-accounts)). I could not confirm from primary sources whether a final rule has been adopted. Treat it as proposed.
- **FDIC insurance does not cover the platform failing.** The same FDIC release says "FDIC insurance does not protect against the default, insolvency, or bankruptcy of any non-bank entity."

### 1c. Payment facilitator (payfac) under a sponsoring acquirer

The platform registers with Visa and Mastercard through an acquirer and signs merchants as *sponsored merchants*.

- Visa defines a PF as a third-party agent that "signs a merchant acceptance contract with a sponsored merchant on behalf of an acquirer" and "receives & distributes settlement of transaction proceeds from an acquirer, on behalf of its sponsored merchant(s)." "Acquirers are responsible for the acts of both PFs and sponsored merchants" ([Visa PF model, Feb 2024](https://usa.visa.com/dam/VCOM/global/support-legal/documents/visa-payment-facilitator-model.pdf)).
- **Obligations.** The acquirer registers the PF and performs due diligence. The PF monitors its sponsored merchants. A sponsored merchant above a network volume threshold may need a direct agreement with the acquirer. Visa's acquirer risk standards put this threshold at US$1M of annual Visa volume, a figure I saw only in secondary summaries of that document ([Visa PF model](https://usa.visa.com/dam/VCOM/global/support-legal/documents/visa-payment-facilitator-model.pdf); [Visa Core Rules](https://usa.visa.com/dam/VCOM/download/about-visa/visa-rules-public.pdf)).
- Becoming a payfac is a card-network role about acquiring and liability. It does not settle the money transmission question. The PF still receives settlement and pays it on, so it still needs a state and federal answer, usually the same agent-of-payee and processor exemptions as 1b. Large payfac-style platforms also commonly hold MTLs, but I did not verify any specific company's licensing.

### 1d. Hybrid ("payfac-as-a-service")

PSPs also sell their own payfac registration to platforms. Stripe markets Connect this way ([Stripe: payment facilitation](https://stripe.com/payment-facilitation)). The network obligations and the funds-holding license stay with the PSP, and the platform owns the merchant relationship and UX. Functionally this is 1a.

## 2. Money transmission framing per arrangement

| Arrangement | Who holds merchant funds | Federal (FinCEN) framing | State framing | What the platform must get right |
|---|---|---|---|---|
| 1a PSP platform (Connect) | Licensed PSP (Stripe Payments Company) | PSP is the registered MSB | PSP holds the MTLs | Charge type and account configuration, which set refund, chargeback and negative-balance liability |
| 1b Platform FBO + sponsor bank | Bank, in a custodial account the platform controls | Payment processor exemption: goods and services, formal agreement with the merchant, in *and out* via BSA-regulated rails | Agent-of-payee exemption where enacted. Otherwise a licence, or rely on state-specific rules | Beneficial-ownership sub-ledger, daily reconciliation, no commingling, ACH-only payouts |
| 1c Payfac | Platform (settlement from acquirer), usually into an FBO account | Same as 1b | Same as 1b. Payfacs often hold MTLs as well | All of 1b plus network registration, sponsored-merchant monitoring and acquirer liability |

### The federal test, in the source's words

[FIN-2014-R009](https://www.fincen.gov/sites/default/files/administrative_ruling/FIN-2014-R009.pdf) (a company that "collects payments on behalf of those merchants, and settles with the merchants") lists four conditions for the payment processor exemption (31 CFR 1010.100(ff)(5)(ii)(B)):

> (a) the entity providing the service must facilitate the purchase of goods or services, or the payment of bills for goods or services (other than money transmission itself);
> (b) the entity must operate through clearance and settlement systems that admit only BSA-regulated financial institutions;
> (c) the entity must provide the service pursuant to a formal agreement; and
> (d) the entity's agreement must be at a minimum with the seller or creditor that provided the goods or services and receives the funds.

On condition (b), FinCEN focused on the *disbursement* side. If payouts happen "outside of a clearance and settlement system that admits only BSA-regulated financial institutions (such as, in the form of monetary instruments)", the exemption is lost.

### The state test, in the source's words

MTMA §2.01(q): "Money transmission" includes "Receiving money for transmission from a person located in this state." MTMA §3.01(b) exempts:

> A person appointed as an agent of a payee to collect and process a payment from a payor to the payee for goods or services, other than money transmission itself, provided to the payor by the payee, provided that: (1) there exists a written agreement between the payee and the agent directing the agent to collect and process payments from payors on the payee's behalf; (2) the payee holds the agent out to the public as accepting payments for goods or services on the payee's behalf; and (3) payment for the goods and services is treated as received by the payee upon receipt by the agent so that the payor's obligation is extinguished and there is no risk of loss to the payor if the agent fails to remit the funds to the payee.

Related MTMA provisions worth knowing ([MTMA PDF](https://www.csbs.org/sites/default/files/2023-02/CSBS%20Money%20Transmission%20Modernization%20Act.pdf)):
- §3.01(a) exempts payment-system operators that clear and settle between exempt or licensed persons.
- §3.01(g) exempts federally insured banks, and §3.01(m) exempts their appointed agents only if the bank "assumes all risk of loss". This is the legal shape of a true sponsor-bank program.
- For licensees, §10.03 requires permissible investments at least equal to outstanding money transmission obligations, held in statutory trust. §10.04(a)(1) counts "funds in such accounts held for the benefit of the licensee's customers in a federally insured depository financial institution" as cash. This is the source of the "FBO balance ≥ obligations" invariant.

State adoption varies. The MTMA is a model law, 31 states had enacted it in whole or in part as of September 2026 ([CSBS](https://www.csbs.org/csbs-money-transmission-modernization-act-mtma)), and agent-of-payee wording differs among the other states. A real launch would need state-by-state counsel. The design doc should say so in one sentence.

## 3. Recommendation for Plumb

**Model 1b: the platform holds merchant funds in an FBO account at a sponsor bank, a PSP acquires card payments and settles into that account, and payouts go out by ACH. Frame the platform as each merchant's agent of payee, operating within the payment processor exemption.** Mention 1a (Stripe Connect) in the doc as the alternative a real early-stage company might buy, and say why Plumb builds 1b: the ledger, reconciliation and payout lifecycle are the point of the portfolio.

Why this reads credibly:
- It is the arrangement map #2 already settled: platform-held FBO account, *merchant payable* split into unsettled and available, ACH-style payouts with returns, and *payouts in transit*. Connect would hide the very ledger Plumb exists to show.
- A B2B invoice platform fits agent of payee closely. There is a written merchant agreement, the platform is held out as collecting for the merchant, and payment is for goods and services.
- Payfac (1c) adds card-network registration and acquirer liability. That is real work but off-thesis, and the design doc can name it as a later-stage option.

### Design-doc hooks this framing implies (candidate invariants, not decisions)

These follow from the sources above. Each is a cheap, checkable property that makes the framing more than words:

1. **Invoice is paid when the platform receives funds.** Agent-of-payee condition (3) says the payer's obligation is extinguished when the agent receives the money. Plumb should mark the merchant's invoice paid once the platform has the payer's funds, whether on capture or on settlement. The payer then never bears the risk of the platform failing to remit.
2. **The FBO account holds only merchant money.** The platform fee moves out to the platform's operating account as a posted entry (a sweep). The risk reserve stays in the FBO account because it is merchant money held back.
3. **FBO cash reconciles to merchant payable every day.** The bank balance plus PSP receivables in transit must equal total merchant payable plus payouts in transit. This mirrors the FDIC proposal's daily per-owner reconciliation and the MTMA §10.03 permissible-investments test. It belongs in the reconciliation job.
4. **The sub-ledger is the beneficial-ownership record.** Per-merchant balances in Plumb's ledger are the "owner of each dollar" record that pass-through insurance and the FDIC proposal rely on.
5. **Payouts go out only over ACH to the merchant's own verified bank account.** FinCEN condition (b) applies to disbursement, and paying third parties or issuing instruments breaks the exemption framing. Out of scope stays out of scope: no wallets, no paying anyone other than the merchant, no stored value.
6. **Negative merchant payable is covered by platform money, never by other merchants' funds.** Covering it from the FBO pool would commingle one merchant's funds with another's. Refunds after payout become a platform receivable from the merchant, recovered from future settlements or an ACH debit.

### Suggested one-paragraph framing for the design doc

> Plumb's platform collects invoice payments as each merchant's agent of payee under a written merchant agreement. A payer's invoice is treated as paid once the platform receives funds. Card payments are acquired by a PSP and settled into an FBO custodial account at a sponsor bank, and payouts go to merchants' own bank accounts over ACH only. This shape is meant to fit the FinCEN payment processor exemption and the agent-of-payee exemption in states that adopted the CSBS MTMA. It is a design assumption, not a legal conclusion: a real launch needs state-by-state counsel, and the common alternative is to buy the funds-holding role from a licensed PSP (for example Stripe Connect).

## Sources

Primary:
- FinCEN, [FIN-2014-R009](https://www.fincen.gov/sites/default/files/administrative_ruling/FIN-2014-R009.pdf), 27 Aug 2014: payment processor exemption, four conditions, disbursement point. Read in full.
- CSBS, [Model Money Transmission Modernization Act](https://www.csbs.org/sites/default/files/2023-02/CSBS%20Money%20Transmission%20Modernization%20Act.pdf): §§2.01(q), 3.01(a)(b)(g)(m), 10.03, 10.04. Read in full.
- CSBS, [MTMA landing page](https://www.csbs.org/csbs-money-transmission-modernization-act-mtma): 31-state adoption count.
- Stripe Docs: [Connect charge types](https://docs.stripe.com/connect/charges), [connected account types](https://docs.stripe.com/connect/accounts), [payouts to connected accounts](https://docs.stripe.com/connect/payouts-connected-accounts).
- Stripe Legal: [Stripe Payments Company licenses](https://stripe.com/legal/spc/licenses), [SPC terms](https://stripe.com/legal/spc). Seen via search snippet, not fetched in full.
- Visa, [Visa's Payment Facilitator Model](https://usa.visa.com/dam/VCOM/global/support-legal/documents/visa-payment-facilitator-model.pdf), Feb 2024. Read in full.
- FDIC, [proposed custodial-account recordkeeping rule press release](https://www.fdic.gov/news/press-releases/2024/fdic-proposes-deposit-insurance-recordkeeping-rule-banks-third-party), Sept 2024; [Federal Register 89 FR 80135](https://www.federalregister.gov/documents/2024/10/02/2024-22565/recordkeeping-for-custodial-accounts).
- FDIC, [Pass-through deposit insurance coverage](https://www.fdic.gov/financial-institution-employees-guide-deposit-insurance/pass-through-deposit-insurance-coverage).

Not verified from primary text: the US$1M sponsored-merchant threshold (Visa Core Rules §5.3, seen only in secondary summaries of Visa's acquirer risk standards), whether the FDIC custodial-account rule has been finalized, and the state-by-state spread of agent-of-payee exemptions outside MTMA states.
