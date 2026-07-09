# Subscription (Buy BRH)

**Ledger:** client · **Portfolios (`CLIENT`):** the client's HTG `INVESTMENT` book (BRH position) and its `CASH` settlement book (cash leg) · **Counterparty:** none (client-facing)

## When it happens

A client invests in a BRH emission that already exists (emissions are created beforehand, when BRH
announces an issue). The operator books the subscription against that emission — there is no
creation step here, the operator picks an existing emission.

## What the operator does

Selects the emission, the client's HTG investment portfolio, and the subscription amount (nominal in HTG), and sets
the **new resource (yes/no)** indicator — whether the money is fresh external funds or a
reallocation of money already under management. The indicator feeds new-money reporting; it does
not change the transaction's economics.

## Transaction produced

A **Buy BRH** transaction across the client's books: it references the BRH emission instrument; the
cash leg debits the client's `CASH` settlement portfolio, and the BRH position opens on the client's
HTG `INVESTMENT` portfolio (BRH bonds are HTG). The position is not stored — it is derived from this
and later transactions.

## Charges

Charges come from the charge matrix and are snapshotted at booking. A BRH subscription carries
**none** — there is no subscription commission for BRH. Any BRH commission applies at **maturity**,
not here.

## Settlement lifecycle

This subscription enters the settlement lifecycle in the **pending** state. It becomes eligible to
be picked into a [Send to BRH](../house/send-to-brh.md) batch, which will move it to **sent**, then
**debited**, then **credited** as the house flows progress.

## Accounting (events named; entries in the Accounting module)

Client cash out, BRH position in.

## Control

Sensitive — subject to maker/checker approval.
