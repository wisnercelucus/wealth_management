# Market Data

The **observation store** — what the market said, per date. Two tables, no
computation: prices don't value portfolios (the Position module does), rates don't
convert anything (the posting engine does). Modules read; this module records.

```
InstrumentPrice   -- one price per instrument per date (securities, fund NAVs, index values)
FxRate            -- one rate per currency pair per date
```

## Who reads it

- **Form pre-fill** — `Transaction.fxRate` defaults from the rate series as-of
  `tradeDate`; the price field pre-fills from the price series. No row for the date →
  nothing pre-fills and the operator supplies the value (the gate stamps deviations on
  override, unchanged).
- **Witness sourcing** — `Lot.fxAtOpen` / `LotRelief.fxAtClose` read the series as-of
  the open / disposal date whenever the transaction itself didn't convert.
- **Valuation** — the Position module prices positions and converts to reference
  currency from these two tables.
- **OBL BRH index compensation** — the benchmark instrument's values live in
  `InstrumentPrice`; the maturity flow compares the issue-date value to the current
  one.

**As-of rule:** "as-of date D" = the most recent row on or before D.

## The one property that matters

**The ledger never re-reads the series.** Every applied value is frozen at posting —
in the header (`fxRate`), the witnesses, `fxAtOpen`/`fxAtClose`, the legs. So a series
row can be corrected at any time without changing any posted truth; only future
pre-fills, valuations, and reports see the correction.

## Shared envelope

Every entity carries the standard envelope — see the
[transactions README](../transactions/README.md#shared-envelope). These are ordinary
mutable reference data: corrections edit rows (`modifiedAt/By` used).
