# InstrumentType

The **seeded catalog of instrument kinds** — what an instrument _is_ (a bond, a share, a
fund, cash…), plus the kind-level parameters the platform computes from. Referenced by
[Instrument](./Instrument.md) (`instrumentTypeId`) and by the transaction domain's
eligibility table (`TransactionTypeEligibility.instrumentTypeId`), which permits
transaction types per kind.

Like the party-role catalog, this list is **seeded and stable**: the code drives code
branching (which type-specific part carries the detail, what the posting gate treats as
cash-class), so a new kind arrives with code, never as free data entry.

> Assumes the shared envelope (see [README](./README.md)).

## Fields

| Field             | Type        | Req | Meaning                                                                                                                                                                                                                                                                                                                         |
| ----------------- | ----------- | --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`              | uuid/bigint | ●   | Surrogate PK.                                                                                                                                                                                                                                                                                                                   |
| `code`            | string      | ●   | Stable kind code — unique. Application logic keys off it.                                                                                                                                                                                                                                                                       |
| `name`            | string      | ●   | Display label.                                                                                                                                                                                                                                                                                                                  |
| `quotationFactor` | decimal(10,6) | ●   | The multiplier that turns `quantity × price` into money: `1` for unit-priced kinds (shares, funds, cash), `0.01` for percent-of-nominal quotation (ordinary bonds: a price of 98.5 means 98.5% of face value). Keeps the transaction engine's `PRINCIPAL = quantity × price × quotationFactor` a single formula for every kind. |
| + envelope        |             | ●   | see README.                                                                                                                                                                                                                                                                                                                     |

## The seeded kinds

| `code`      | `quotationFactor` | Type-specific part                                                                 |
| ----------- | ----------------- | ---------------------------------------------------------------------------------- |
| `BOND`      | 0.01              | ordinary bonds — part to be designed: coupon rate, schedule, day-count             |
| `EQUITY`    | 1                 | shares — part to be designed; includes the preferred-share behaviors               |
| `FUND`      | 1                 | funds — part to be designed                                                        |
| `INDEX`     | 1                 | benchmark / reference series: never held, never transacted (no eligibility row may target INDEX)                         |
| `CASH`      | 1                 | none — the base suffices (Cash-HTG, Cash-USD; several per currency allowed for special purposes: each cash-holding book names its own via `Portfolio.cashInstrumentId`) |
| `OBL_BRH`   | 1                 | [OblBrh](./OBL%20BRH/OblBrh.md) — nominal entered directly, price 1                |
| `PROVISION` | 1                 | none — cash-like holding instruments: price constant 1, currency-bound, never lots |

## Notes

- **Kind vs class.** The kind says _what an instrument is_ (this table); its
  [AssetClass](./AssetClass.md) says _how it behaves_ for reporting and AUM. Eligibility
  works at kind granularity because lifecycle law is kind-shaped ("OBL BRH has no
  secondary market").
- **Calculators.** The amounts a kind can compute (GROSS, BASE_INTEREST, …) are
  registered in [InstrumentCalculator](./InstrumentCalculator.md); the transaction
  type-authoring UI reads that registry to offer only computable amounts.
- **Provision seeds.** Two instruments carry the `PROVISION` kind: `PROV-HTG` and
  `PROV-USD` (asset class `PROVISION`, issuer ProFin). They hold money that is reserved
  or awaiting settlement, so cash availability reads correctly with no extra logic; the
  provision report groups their positions by transaction type.

## Clean model

```
InstrumentType
  id               uuid    PK
  code             string  unique    -- BOND, EQUITY, FUND, INDEX, CASH, OBL_BRH, PROVISION
  name             string
  quotationFactor  decimal(10,6)     -- 0.01 percent-quoted bonds; 1 otherwise
  + envelope
```
