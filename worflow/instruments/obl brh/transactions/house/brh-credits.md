# BRH Credit (maturity / early redemption)

**Ledger:** house · **Portfolio:** ProFin's BRH-account portfolio (`OPERATIONAL`) ·
**Counterparty:** BRH · **Charges:** none

## When it happens

BRH credits ProFin's BRH account when an emission ends. Two cases:

- **Maturity credit** — at term, BRH credits the **proceeds** (capital + interest). The prime de
  succès, credited earlier at the [Debit Note](./debit-note.md), **stays earned**.
- **Early credit note** — on early redemption, BRH credits the proceeds **less the reclaimed
  prime**. The prime credited at the debit note is taken back, so ProFin receives **less than the
  full amount**. This reclaim is what the client sees as the early-redemption penalty.

## Transaction produced

A **BRH Credit** transaction (cash in) on ProFin's BRH-account portfolio, counterparty BRH —
maturity proceeds, or early-redemption proceeds net of the reclaimed prime.

## Settlement lifecycle

Moves the covered subscriptions to **credited**, reconciled through the batch / allocation so each
credit ties back to the subscriptions and emissions it settles — including when one credit covers
subscriptions sent across several batches.

## Accounting (events named; entries in the Accounting module)

Cash in; at maturity the proceeds with the prime retained; on early redemption the proceeds net of
the reclaimed prime.
