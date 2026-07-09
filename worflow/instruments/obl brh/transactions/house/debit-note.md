# BRH Debit Note (Note de débit)

**Ledger:** house · **Portfolio:** ProFin's BRH-account portfolio (`OPERATIONAL`) ·
**Counterparty:** BRH · **Charges:** none

## When it happens

BRH acts on ProFin's BRH account for the batched subscriptions: it **debits the principal** (the
money taken up against the bonds) and **credits the prime de succès** (ProFin's earning). Both
happen at the debit note.

## Transaction produced

A **BRH Debit Note** transaction on ProFin's BRH-account portfolio, counterparty BRH. Net effect on
the account: principal out, prime in.

## Settlement lifecycle

Moves the covered subscriptions from **sent** to **debited**, reconciled through the batch recorded
by [Send to BRH](./send-to-brh.md).

## Accounting (events named; entries in the Accounting module)

The principal placed in BRH bonds, and the prime de succès earned. These are the two economic
components the accounting design turns into journal lines.
