# InstrumentPrice

One **price observation** per instrument per date — securities, fund NAVs, and
index/benchmark values alike (an index value *is* a price; the BRH index series lives
here, read by the OBL BRH compensation).

## Essential fields

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `instrumentId` | FK→Instrument | ● | The instrument observed — any kind, including INDEX. |
| `priceDate` | date | ● | Observation date. |
| `price` | decimal(28,12) | ● | In the instrument's own currency and quotation convention — percent-quoted bonds store the par-relative price; the kind's `quotationFactor` applies at use, exactly as in the posting engine. |
| + envelope | | ● | See [README](README.md). Provenance rides the envelope (`createdBy` = the importer or the operator, `externalId` / `externalRef` = the external feed's reference). |

`UNIQUE (instrumentId, priceDate)` where `isActive`.

## Notes & rules

- **Fund NAVs are published here** by the fonds module once computed — consumers never
  care who produced a price.
- Corrections edit the row; nothing posted ever re-reads the series (see
  [README](README.md)).
- Index instruments are never transacted ([eligibility
  rule](../transactions/TransactionTypeEligibility.md)) — their rows exist purely to
  be read as a series.

## Deliberately absent

- **Currency** — the instrument's own; nothing here is posted truth, so the snapshot
  argument doesn't apply.
- **A current-price table** — "current" = the latest row on or before today: a
  derivable, never stored.
- **Bid/ask, intraday times, price type** — no MVP consumer; one observation per day.

## Clean model

```
InstrumentPrice
  id            uuid    PK
  instrumentId  FK Instrument
  priceDate     date
  price         decimal(28,12)   -- instrument's currency & quotation convention
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)
  unique (instrumentId, priceDate) where isActive
```
