# BankAccount

ProFin's **own** bank accounts — the omnibus / operational accounts it holds at banks. Each FK to a
bank [Party](Party.md) (the `BANQUE` role). The settlement reference, not custody. Per-client cash
is **not** here — that's the client's cash portfolio.

> Assumes the shared envelope (see [README](README.md)).

## Essential fields

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid/bigint | ● | Surrogate PK |
| `bankPartyId` | FK→Party | ● | The bank (BANQUE role) |
| `currencyId` | FK→Currency | ● | Account currency |
| `accountName` | string | ● | Account label (e.g. `BNC Opérations PRF USD`) |
| `code` | string(50) | ● | Account code — unique |
| `purpose` | enum | ○ | `OPERATIONS \| PLACEMENTS \| FOREX \| TRESORERIE` |
| `accountNumber` | string | ○ | Bank account number |
| + envelope | | ● | see README |

## Notes & rules

- **ProFin's own accounts only.** Client deposits land in these omnibus accounts; each client's
  balance is tracked in their cash portfolio, not here.
- The bank itself is a `BANQUE`-role Party — `bankPartyId` points at it.
- **GL linkage lives in the accounting module** — the account ↔ chart-of-accounts mapping is held
  there, not as a field on this table.
- **Which portfolio maps to it** — a ProFin house [Portfolio](../portefeuilles/Portfolio.md) points
  here via its optional `bankAccountId`; the link is held on the portfolio, not back-referenced here.

## Clean model

```
BankAccount
  id            uuid    PK
  bankPartyId   FK Party              -- BANQUE role
  currencyId    FK Currency
  accountName   string
  code          string  unique
  purpose       enum (OPERATIONS | PLACEMENTS | FOREX | TRESORERIE)?
  accountNumber string?
  + envelope
```
