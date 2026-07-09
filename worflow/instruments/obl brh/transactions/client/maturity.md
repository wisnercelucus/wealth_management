# Maturity

**Ledger:** client · **Portfolios (`CLIENT`):** the client's HTG `INVESTMENT` book (BRH position) and its `CASH` settlement book (proceeds) · **Counterparty:** none (client-facing)

## When it happens

The emission reaches its maturity date. The system **auto-generates** the maturity operation (a
daily-maturities process); the operator **validates** it rather than entering it from scratch.

## What is paid

The payout is computed, not stored:

- **Capital** — the subscribed nominal.
- **Interest** — the Custom-BRH fixed interest (`annualRate ÷ rateDivisor × days`).
- **Benchmark compensation (the spread)** — added only if the benchmark has moved unfavourably:
  the difference between the benchmark's current value and its value on the emission's issue date
  (read live from the series). If it has not moved unfavourably, the fixed rate stands alone.

## Transaction produced

A **Maturity** transaction across the client's books: the BRH position closes on the HTG `INVESTMENT`
portfolio, and the payout is credited to the `CASH` settlement portfolio.

## Charges

Charges come from the charge matrix and are snapshotted at booking. OBL BRH's one charge is the
**ProFin commission** taken here, at maturity: a percentage of the fixed interest, by the client's
tariff category, plus the **TCA** on that commission. It will be configured in the charge module
(not yet designed), and never applies at subscription. **OBL BRH has no withholding.**

## Settlement lifecycle

The maturity payout corresponds to the **BRH Credit** (maturity proceeds) on the house ledger; the
prime de succès stays earned. The subscription's lifecycle reaches **credited** when that house
credit is reconciled to it.

## Accounting (events named; entries in the Accounting module)

Position closed, client cash in; interest recognized.

## Control

Auto-generated, operator-validated.
