# Plumb

Payment orchestration, double-entry ledger and real-time risk decisioning for a B2B billing platform that collects invoice payments from payers on behalf of merchants.

## Language

### Parties

**Platform**:
The business that operates Plumb, runs the payment rails and earns a fee on each payment.
_Avoid_: Operator, Plumb (when meaning the business)

**Merchant**:
A business that sends invoices through the platform and receives payouts of the funds collected for it.
_Avoid_: Seller, client, customer

**Payer**:
A business that pays a merchant's invoice through the platform.
_Avoid_: Customer, buyer, cardholder

### Money held

**FBO account**:
The "for benefit of" bank account where funds owed to merchants are held, kept separate from the platform's own money.
_Avoid_: Platform cash, operating account

**Merchant payable**:
The amount the platform owes a merchant, split into an unsettled balance and an available balance. It can go negative when refunds arrive after a payout.
_Avoid_: Merchant balance, wallet

**Risk reserve**:
Funds held back from a merchant's payouts to cover later refunds and chargebacks.
_Avoid_: Rolling reserve, holdback

**Payout**:
A transfer of a merchant's available funds from the FBO account to the merchant's bank account.
_Avoid_: Settlement, disbursement

**Bank rail**:
The ACH-style bank transfer network that carries payouts from the FBO account to merchants' bank accounts; simulated in Plumb.
_Avoid_: Payout rail, bank network

**Payouts in transit**:
Funds already taken out of merchant payable for a payout that the bank has not yet confirmed as paid.
_Avoid_: Pending payouts, outgoing

**Payout return**:
A bank's rejection of a payout days after it was sent, which puts the funds back into merchant payable.
_Avoid_: Bounce, failed payout

**Unsettled balance**:
The part of a merchant's payable from captured payments that the PSP has not yet settled into the FBO account.
_Avoid_: Pending balance

**Available balance**:
The part of a merchant's payable from settled funds, and the only part eligible for payout.
_Avoid_: Withdrawable balance, cleared balance, pending balance

**Payout hold**:
A block on a merchant's payouts, placed by the merchant risk policy, that keeps the available balance in the FBO account.
_Avoid_: Freeze, payout block

### Ledger

**Pending entry**:
A ledger entry that records an authorization hold. It is never edited and counts toward no balance a merchant can be paid from.
_Avoid_: Hold entry, provisional entry

**Posted entry**:
A final ledger entry, written at capture or at any later money movement, that changes balances.
_Avoid_: Committed entry, settled entry

**PSP clearing**:
The ledger account for what the PSP owes the platform: captures net of PSP fees and refunds, not yet settled into the FBO account. Reconciliation checks it against the PSP's own report.
_Avoid_: PSP receivable, customer receivable

**Margin sweep**:
The move of the platform's own share of a settlement (platform fee minus PSP fee) out of the FBO account into the platform's operating cash, posted with each settlement so the FBO account holds only merchant money.
_Avoid_: Fee withdrawal, revenue transfer

### Risk

**Merchant risk policy**:
The deterministic rules that set each merchant's risk reserve percentage and place payout holds, based on the merchant's refund rate, chargeback rate and account age.
_Avoid_: Merchant scoring, merchant model, underwriting model

**Merchant tier**:
The class a merchant is placed in, new or established, that sets its default payout cadence, delay after capture and risk reserve. The merchant risk policy moves merchants between tiers.
_Avoid_: Merchant level, risk band, merchant plan
