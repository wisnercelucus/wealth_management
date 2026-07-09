# FxRate

One **rate observation** per currency pair per date. The pair's definition (base /
quote) lives in [CurrencyPair](../../references/CurrencyPair.md); this table holds the
values.

## Essential fields

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `currencyPairId` | FK→CurrencyPair | ● | The pair observed. |
| `rateDate` | date | ● | Observation date. |
| `rate` | decimal(28,12) | ● | In the pair's own **base → quote** convention. Consumers derive the direction they need — `Transaction.fxRate` is always quoted instrument → settlement, so the pre-fill inverts when the pair is stored the other way. |
| + envelope | | ● | See [README](README.md). Provenance rides the envelope. |

`UNIQUE (currencyPairId, rateDate)` where `isActive`.

## Notes & rules

- One rate per pair per date — no bid/ask, no intraday (no MVP consumer).
- Pre-fill with no row as-of the date leaves the field empty; the operator supplies
  the rate and the gate stamps the deviation as usual.
- Corrections edit the row; every applied rate is already frozen in posted truth
  (header `fxRate`, `COUNTER`, `fxAtOpen`, `fxAtClose`) — see [README](README.md).

## Clean model

```
FxRate
  id              uuid    PK
  currencyPairId  FK CurrencyPair
  rateDate        date
  rate            decimal(28,12)   -- the pair's base → quote convention
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)
  unique (currencyPairId, rateDate) where isActive
```
