# Repatriation (Rapatriement)

**Ledger:** house (internal transfer) · **Portfolios:** ProFin BRH account → ProFin commercial-bank
operational account (both `OPERATIONAL`) · **Counterparty:** none (both legs are ProFin's) · **Charges:** none

## When it happens

ProFin moves funds standing in its BRH account back to its commercial-bank operational account (e.g. SGBK).

## Transaction produced

**One** transaction with **two legs** (double-entry): a debit on the BRH-account portfolio and a
credit on the commercial-bank operational portfolio. The legs net to zero (balance check). No counterparty, since both
sides are ProFin's own books.

## Accounting (events named; entries in the Accounting module)

A movement between two house cash books.
