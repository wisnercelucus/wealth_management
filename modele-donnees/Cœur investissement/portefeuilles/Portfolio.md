# Portfolio

The **account** the aggregate root of the domain. Holds securities and cash for one or
more parties, in a base currency, under a mandate. Tax and commissions are **not** here;
they live in the taxation layer.

## Essential fields

| Field                       | Type                | Req | Description                                                                                                                                                                                                                                                                                    |
| --------------------------- | ------------------- | --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                        | uuid                | ●   | Surrogate primary key.                                                                                                                                                                                                                                                                         |
| `code`                      | string(50)          | ●   | Account code — the unique business key. Join on `id`, not this; enforce a clean canonical value (no stray whitespace).                                                                                                                                                                         |
| `name`                      | string              | ●   | Portfolio name.                                                                                                                                                                                                                                                                                |
| `currencyId`                | FK→Currency         | ●   | Base / reporting currency (HTG or USD).                                                                                                                                                                                                                                                        |
| `cloisonnementEntityId`     | FK→Entity           | ●   | Segregation unit (security layer). Defaults from the client's `PartyProfile.cloisonnementEntityId`; **authoritative** grain for what a user sees.                                                                                                                                              |
| `benchmarkInstrumentId`     | FK→Instrument       | ○   | Performance benchmark — an `indice` / benchmark instrument. Used by reporting (P&L vs benchmark).                                                                                                                                                                                              |
| `costBasisMethod`           | enum                | ○   | Cost-basis method for realized P&L: `FIFO \| AVERAGE_COST`. Investment portfolios; default `FIFO`, the only method implemented in the MVP (`AVERAGE_COST` is a reserved future walk strategy). `FUND` books require FIFO: exit fees read the age of the specific units redeemed.                                                                                                                                                                             |
| `serviceType`               | enum                | ●   | Mandate: `EXECUTION_ONLY \| ADVISORY \| DISCRETIONARY \| CUSTODY`. Descriptive — it gates advisory / discretionary behaviour that is out of MVP scope.                                                                                                                                         |
| `portfolioType`             | enum                | ○   | Structural type: `INVESTMENT \| CASH`. Determines whether a portfolio is investment or cash. `NULL` when `bookKind ≠ CLIENT` (ProFin's own house books have no investment-vs-cash structure).                                                                                                  |
| `bookKind`                  | enum                | ●   | Which kind of book this is: `CLIENT \| FUND \| ISSUER \| TREASURY \| OPERATIONAL`. **Only `CLIENT`** is a real client portfolio and the only kind that counts toward AUM. `FUND` is an in-house fund's own book (NAOS): it holds the fund's assets and is excluded from AUM because the clients' fund units in their `CLIENT` books already carry that value (counting both would double-count the same money). `ISSUER`, `TREASURY`, and `OPERATIONAL` are ProFin's own house books, excluded from every AUM calculation and client report. |
| `bankAccountId`             | FK→BankAccount      | ○   | ProFin's own bank account this book maps to / settles through (see [BankAccount](../clients/BankAccount.md)). Set on ProFin's house books (`ISSUER` / `TREASURY` / `OPERATIONAL`); `NULL` on `CLIENT` books — per-client cash lives in the client's `CASH` portfolio, not a ProFin bank account. At most one book per bank account (partial `UNIQUE (bankAccountId)` where not null): a bank account picked by a portfolio cannot be picked by another, so the ledger's `TRANSACTION_BANK` mode always resolves a bank account to a single house book. |
| `status`                    | enum                | ●   | Business lifecycle: `ACTIVE \| BLOCKED \| INACTIVE \| CLOSED` _(Actif / Bloqué / Inactif / Fermé)_.                                                                                                                                                                                            |
| `closureDate`               | date                | ○   | Set when `status = CLOSED`.                                                                                                                                                                                                                                                                    |
| `cashSettlementPortfolioId` | FK→Portfolio (self) | ○   | The cash account that settles this one. On client books, `NULL` ⇒ this _is_ a cash account. `FUND` and house books leave it `NULL` and **self-settle**: one book holds securities and cash, so their cash legs land in the book itself.                                                                                                                                                                                                                     |
| `cashInstrumentId`          | FK→Instrument       | ○   | The cash instrument this book's money moves through (CASH kind, gate-validated). Required when `portfolioType = CASH`, on `FUND` books (they are their own settlement cash, so their money — including scheduled charges — moves through it), and on house books carrying a `bankAccountId`; `NULL` otherwise. This is what the ledger's automatic resolutions read: the settlement default on trades and the cash instrument of `TRANSACTION_BANK` legs. Several cash instruments can exist per currency; the book says which one it uses. On a bank-linked house book its currency must match the bank account's (gate-validated).                                              |
| `openingDate`               | date                | ●   | Account opening date.                                                                                                                                                                                                                                                                          |
| `closureReason`             | string              | ○   | Free-text reason on closure.                                                                                                                                                                                                                                                                   |
| + envelope                  |                     | ●   | See [README](README.md): `isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy`.                                                                                                                                                                                    |

## Cash vs investment

`portfolioType` is the source of truth. By convention an `INVESTMENT` portfolio sets
`cashSettlementPortfolioId` to a `CASH` portfolio, and a `CASH` portfolio leaves it null —
but the **type**, not the presence of the link, is what defines the structure. Never infer
it from the `code` suffix (`- C` / `- HTG`); that convention is unenforced and unreliable.
`portfolioType` is set only on client books (`bookKind = CLIENT`); it is `NULL` on fund books and
ProFin's own house books (`FUND` / `ISSUER` / `TREASURY` / `OPERATIONAL`), which carry no
investment-vs-cash structure.

## Lifecycle & closure

- `status` is the business state the user sees and acts on: close, block, reactivate.
- Closing is a transition to `CLOSED` that stamps `closureDate`.
- `status` is distinct from the envelope's `isActive` (row-level soft-delete). A closed
  portfolio is `status = CLOSED` with `isActive = true` — retained for audit and
  historical reporting, never hard-deleted.

## Notes & rules

- A portfolio has **no `clientId`**. Ownership and all party relationships are carried by
  [PortfolioParty](PortfolioParty.md) (many-to-many).
- Tax facts and fees are out of scope here see the taxation layer.
- **Segregation (cloisonnement):** `cloisonnementEntityId` defaults from the client's
  `PartyProfile.cloisonnementEntityId` but can be overridden per portfolio; the security layer
  segregates on this field — the authoritative grain.
- **Cost basis:** `costBasisMethod` drives how the disposal walk consumes lots; default `FIFO`,
  the only method implemented in the MVP. The method is a **walk strategy, never a data shape**:
  lots always store the full acquisition history, so `AVERAGE_COST` (pro-rata consumption), LIFO
  or a custom order can be added later as code, with no schema change. Switching a portfolio's
  method applies to future disposals only (posted reliefs are frozen truth). Not meaningful for
  `CASH` portfolios.
- **Book-kind & AUM (`bookKind`):** the authoritative, book-level driver of AUM inclusion.
  The kinds are `CLIENT` (a real client portfolio), `FUND` (an in-house fund's own book, e.g.
  NAOS: it holds the fund's assets), `ISSUER` (ProFin's issuer premiums),
  `TREASURY` (ProFin's treasury reserves), and `OPERATIONAL` (operational working books — ProFin's
  operational bank accounts such as its commercial-bank operational account and the BRH settlement
  account, plus provisions and tracking fees). **Only `CLIENT` counts toward AUM**: `FUND`
  books are excluded because the clients' fund units in their `CLIENT` books already carry
  that value (counting both would double-count the same money), and `ISSUER` / `TREASURY` /
  `OPERATIONAL` are ProFin's own house books, excluded from every AUM calculation and client
  report. AUM inclusion is decided by **book-kind, not party role**: a single party (e.g.
  ProFin) can be broker, custodian, issuer _and_ hold its own books at once, so a party role
  cannot express "is this an AUM book" — `bookKind` is the home for that distinction. It
  governs `portfolioType`: the structural type is set only when `bookKind = CLIENT` and is
  `NULL` otherwise. The **full AUM filter** is `bookKind = CLIENT` intersected with an
  instrument-level AUM-eligibility flag defined in the instrument / asset-class layer
  (specified separately).
- **Bank-account linkage (`bankAccountId`):** optional FK to the [BankAccount](../clients/BankAccount.md)
  a ProFin house book maps to / settles through — e.g. the commercial-bank operational account or the
  BRH account in the OBL BRH flow. `CLIENT` portfolios leave it `NULL`: per-client cash is tracked in
  the client's `CASH` portfolio, not a ProFin bank account.

## Clean model

```
Portfolio
  id                         uuid    PK
  code                       string  unique
  name                       string
  currencyId                 FK Currency
  cloisonnementEntityId      FK Entity                            -- defaults from client; authoritative for segregation
  benchmarkInstrumentId      FK Instrument?
  costBasisMethod            enum (FIFO | AVERAGE_COST)?    -- investment; default FIFO; MVP implements FIFO only; FUND books: FIFO
  serviceType                enum (EXECUTION_ONLY | ADVISORY | DISCRETIONARY | CUSTODY)
  portfolioType              enum (INVESTMENT | CASH)?       -- authoritative for client books; NULL when bookKind != CLIENT
  bookKind                   enum (CLIENT | FUND | ISSUER | TREASURY | OPERATIONAL)   -- AUM counts CLIENT only; authoritative
  bankAccountId              FK BankAccount?   -- ProFin's own bank account this book maps to; NULL on CLIENT books; unique where not null (one book per bank account)
  status                     enum (ACTIVE | BLOCKED | INACTIVE | CLOSED)
  closureDate                date?
  closureReason              string?
  cashSettlementPortfolioId  FK Portfolio?   (self)
  cashInstrumentId           FK Instrument?  -- CASH kind; required on CASH books, FUND books, and bank-linked house books
  openingDate                date
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)
```
