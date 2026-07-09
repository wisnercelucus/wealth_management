# Send to BRH (Envoi BRH)

**Ledger:** house (internal transfer) · **Portfolios:** ProFin commercial-bank operational account → ProFin BRH account (both
`OPERATIONAL`) · **Counterparty:** none (both legs are ProFin's) · **Charges:** none

## When it happens

Periodically, ProFin funds its account at BRH with the money for accumulated subscriptions, so BRH
can then debit it. Clients subscribe over time (client ledger); this is the operation that funds a
batch of them.

## What the operator does

Opens a screen listing the BRH subscriptions not yet sent, **selects a set** of them, and triggers
**send to BRH**.

## Transaction produced

**One** transaction with **two legs** (double-entry): a debit on ProFin's commercial-bank operational portfolio and a
credit on ProFin's BRH-account portfolio, for the **sum of the selected subscriptions' amounts**.
The two legs net to zero (balance check). It is a single aggregate movement, not one transaction
per subscription.

## Batch & allocation

The operation records a **batch** with one allocation line per selected subscription, summing to
the transaction total. The batch is the link that lets later BRH payments — which arrive batched on
their own schedule — be reconciled back to the individual subscriptions.

## Settlement lifecycle

Moves each selected subscription from **pending** to **sent**, recording which batch carried it.

## Accounting (events named; entries in the Accounting module)

Cash moves from ProFin's commercial-bank operational book to its BRH-account book.
