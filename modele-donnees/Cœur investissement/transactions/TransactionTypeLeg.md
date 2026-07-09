# TransactionTypeLeg

The **contract lines** of a type — which legs it produces, resolved from which
addresses, fed by which amounts. These lines **are the program** the generic engine runs
for **every** type: each `amountName` fetches its computed
[TransactionAmount](TransactionAmount.md) row and the leg is built from it — no code
ever builds legs. The UI reads them to render the type's form (which inputs to ask
for), and a reviewer reads them instead of code.

## Essential fields

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `transactionTypeId` | FK→TransactionType | ● | The owning type. Frozen with it once transactions exist. |
| `sequence` | int | ● | Execution order. Unique per type. |
| `lotEffect` | enum | ● | `ESTABLISH \| RELIEVE_REALIZE \| CARRY \| ADJUST_BASIS \| RESTATE` — stamped onto the produced leg. See [TransactionLeg](TransactionLeg.md). |
| `principalFlag` | enum | ○ | `PRINCIPAL \| OTHER` — stamped onto the produced leg. Same grammar: only on `CARRY` lines. |
| `portfolioMode` | enum | ● | **Who supplies this leg's portfolio** — the seven modes below. |
| `portfolioId` | FK→Portfolio | ○ | `CHECK: NOT NULL ⟺ portfolioMode = SPECIFIC`. |
| `instrumentMode` | enum | ● | `TRANSACTION \| SPECIFIC \| CASH` — the deal's instrument, a pinned one, or the settlement cash instrument resolved at posting: the header's `settlementCashInstrumentId` (default: the settlement book's `Portfolio.cashInstrumentId`) on trade and `TRANSACTION_INSTRUMENT_ISSUER` legs, the bank house book's `cashInstrumentId` on `TRANSACTION_BANK` legs. Pure cash types don't need it — they pick the cash instrument as the deal's instrument and use mode `TRANSACTION`. |
| `instrumentId` | FK→Instrument | ○ | `CHECK: NOT NULL ⟺ instrumentMode = SPECIFIC`. |
| `direction` | enum | ● | `+ \| −` — the sign applied to the amount. |
| `amountName` | enum | ● | Which named amount feeds this leg (nine-name enum). **Header-derived** names (`QUANTITY`, `PRINCIPAL`, `ACCRUED`, `COUNTER`) are computed generically from the header; every other name is **instrument-derived**, computed at stage 1 by the eligible instrument type's [calculator](../instruments/InstrumentCalculator.md). A line naming an amount its eligible instruments cannot compute is rejected at type-creation. |
| `isConditional` | bool | ● | Whether this leg may legitimately be absent: the line is emitted only when its amount resolves to a non-zero / applicable value (a bond bought on coupon date has no accrued leg). |
| `conditionDescription` | string | ○ | Human note — `"si accrued > 0"`. Prose, **never parsed**. |
| + envelope | | ● | See [README](README.md). |

## The seven portfolio modes

The triplet that wires everything: **header column (value) · mode (pointer) · leg
(result)**. A contract line names which header column it reads; the executor fetches
that column and stamps the leg.

| Mode | Reads | Resolves to |
|---|---|---|
| `TRANSACTION` | header `portfolioId` | the deal's own book |
| `TRANSACTION_CASH` | header `portfolioId` | its linked settlement book (`Portfolio.cashSettlementPortfolioId`); when the deal book has none (`FUND` and house books self-settle), the deal book itself |
| `TRANSACTION_BANK` | header `bankAccountId` | **the** house book carrying that bank account (1-1: `UNIQUE(Portfolio.bankAccountId)`; zero matches → reject, never a silent fallback) |
| `TRANSACTION_COUNTERPARTY` | header `counterpartyPortfolioId` | the chosen counterparty book |
| `TRANSACTION_COUNTERPARTY_CASH` | header `counterpartyPortfolioId` | the counterparty book's linked settlement book (same `cashSettlementPortfolioId` link `TRANSACTION_CASH` follows). Needed the moment an inter-client securities trade settles: the seller's cash leg has no other home. Reads an existing column — no new resolution source. |
| `TRANSACTION_INSTRUMENT_ISSUER` | header `instrumentId` | the issuer book the deal's instrument declares (`Instrument.issuerPortfolioId`). An issuer can run several issuer books (one per product family), so the instrument names its own; `NULL` → reject at entry. Tracks the money on the issuer side (e.g. a bond buy: − client cash, + issuer book). Reads an existing column — no new resolution source |
| `SPECIFIC` | this line's `portfolioId` | pinned — the NAOS fund's own book, a house income book: things that are genuinely type-invariant |

## Growth & authoring rules

- **A new leg = a contract line** (data, free). **A new resolution source = a header
  column + a mode** (code, rare, deliberate — two earned it across the whole MVP:
  `bankAccountId`, `counterpartyPortfolioId`). **Followers are cheaper**: modes that
  read an existing column and walk one of its links (`_CASH` follows
  `cashSettlementPortfolioId`; `TRANSACTION_INSTRUMENT_ISSUER` follows
  `Instrument.issuerPortfolioId`) add no source. The mode set is therefore generated, not
  listed: *each source column × its available links, pruned of degenerate combinations*
  (counterparty × `bankAccountId` collapsed into `TRANSACTION_COUNTERPARTY`: one book
  per bank account makes that follower a round trip) — which is why it closes at seven.
  The bar for an eighth: a genuinely new answer to "who supplies this leg's portfolio,"
  never a new label over an existing source.
- **Exhaust `portfolioId` first.** `TRANSACTION_COUNTERPARTY` is reserved for types
  where two *independent* party books are genuinely in play (client transfer; future
  inter-client trade). ProFin's own omnibus move is `TRANSACTION` (source book picked
  as the deal's portfolio) + `TRANSACTION_BANK` (destination) — every transaction keeps
  a subject.

## Worked contract examples

```
CASH_DEPOSIT                           -- header instrument = the cash instrument
  1  CARRY  ·  TRANSACTION       ·  TRANSACTION  ·  +  ·  PRINCIPAL
  2  CARRY  ·  TRANSACTION_BANK  ·  TRANSACTION  ·  +  ·  PRINCIPAL

CASH_TRANSFER                          -- header instrument = the cash instrument
  1  CARRY  ·  TRANSACTION               ·  TRANSACTION  ·  −  ·  PRINCIPAL
  2  CARRY  ·  TRANSACTION_COUNTERPARTY  ·  TRANSACTION  ·  +  ·  PRINCIPAL

BUY_BOND
  1  ESTABLISH        ·  TRANSACTION       ·  TRANSACTION  ·  +  ·  QUANTITY
  2  CARRY PRINCIPAL  ·  TRANSACTION_CASH  ·  CASH         ·  −  ·  PRINCIPAL
  3  CARRY OTHER      ·  TRANSACTION_CASH  ·  CASH         ·  −  ·  ACCRUED   (conditional: si accrued > 0)

BOND_COUPON
  1  CARRY  ·  TRANSACTION_CASH  ·  CASH  ·  +  ·  INCOME

SECURITIES_TRANSFER
  1  RESTATE  ·  TRANSACTION               ·  TRANSACTION  ·  −  ·  QUANTITY
  2  RESTATE  ·  TRANSACTION_COUNTERPARTY  ·  TRANSACTION  ·  +  ·  QUANTITY

FX_BUY                                 -- header instrument = Cash-USD · settlement = Cash-HTG · fxRate
  1  CARRY  ·  TRANSACTION  ·  TRANSACTION  ·  +  ·  PRINCIPAL
  2  CARRY  ·  TRANSACTION  ·  CASH         ·  −  ·  COUNTER
```

## Clean model

```
TransactionTypeLeg
  id                    uuid    PK
  transactionTypeId     FK TransactionType
  sequence              int              -- unique per type
  lotEffect             enum (ESTABLISH | RELIEVE_REALIZE | CARRY | ADJUST_BASIS | RESTATE)
  principalFlag         enum (PRINCIPAL | OTHER)?    -- only on CARRY lines
  portfolioMode         enum (TRANSACTION | TRANSACTION_CASH | TRANSACTION_BANK |
                              TRANSACTION_COUNTERPARTY | TRANSACTION_COUNTERPARTY_CASH |
                              TRANSACTION_INSTRUMENT_ISSUER | SPECIFIC)
  portfolioId           FK Portfolio?    -- NOT NULL iff SPECIFIC
  instrumentMode        enum (TRANSACTION | SPECIFIC | CASH)
  instrumentId          FK Instrument?   -- NOT NULL iff SPECIFIC
  direction             enum (+ | −)
  amountName            enum             -- nine-name enum; header-derived or instrument-derived (calculator)
  isConditional         bool             -- leg emitted only when its amount applies
  conditionDescription  string?          -- prose, never parsed
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)
  unique (transactionTypeId, sequence)
```
