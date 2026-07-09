# Early redemption (BRH Early Redemption)

**Ledger:** client · **Portfolios (`CLIENT`):** the client's HTG `INVESTMENT` book (BRH position) and its `CASH` settlement book (net payout) · **Counterparty:** none (client-facing)

## When it happens

A client redeems a BRH bond **before** maturity. There is no secondary market — early redemption is
the only exit before term. It is allowed only after the **minimum redemption delay**
(`minRedemptionDelayDays`) has passed; if it has not, the operation is **blocked or warned** (the
behaviour is a configurable setting).

## What is paid

- **Capital** — the nominal.
- **Accrued interest at the fixed rate only** — the benchmark spread is **not** applied on early
  redemption.
- **Less the penalty** — the prorated prime de succès (`successPremiumRate` prorated for the time
  elapsed). The penalty *is* this premium; there is no separate penalty rate.

## Transaction produced

A **BRH Early Redemption** transaction across the client's books: the BRH position closes on the HTG
`INVESTMENT` portfolio, and the net payout is credited to the `CASH` settlement portfolio.

## Charges

Charges come from the charge matrix; an early redemption carries none. The **penalty is
not a charge** — it is the prime de succès, which BRH **reclaims** on the house side: the prime was
credited to ProFin at the [Debit Note](../house/debit-note.md), and BRH's
[early credit note](../house/brh-credits.md) takes it back, so ProFin receives less than the full
proceeds. The client's penalty mirrors that reclaim.

## Settlement lifecycle

Reconciled against BRH's early **credit note** on the house ledger (which reclaims the prime). The
subscription's lifecycle reaches **credited** on that event.

## Accounting (events named; entries in the Accounting module)

Position closed, client cash in; the prime reclaimed.

## Control

Sensitive — maker/checker; subject to the minimum-delay gate.
