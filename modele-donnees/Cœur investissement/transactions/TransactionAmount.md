# TransactionAmount

The **computed witnesses** — one row per named amount the posting computed and froze.
The event's intrinsic anatomy, stored once so that configuration can bind to it **by
name**: charge policies (`basis = PRINCIPAL`), journal templates (`CR revenue ← GROSS`),
leg provenance, statement composition, audit.

## Essential fields

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `transactionId` | FK→Transaction | ● | The event this witnesses. |
| `name` | enum | ● | One of the **nine names** below. Closed set — a tenth name is a code change. |
| `value` | decimal(28,8) | ● | Rounded once, at computation, to currency minor units. |
| `currency` | char(3) | ○ | Per-row — an FX deal holds `PRINCIPAL` in HTG and `COUNTER` in USD. **`NULL` iff `name = QUANTITY`** (a quantity has no currency). |
| + envelope | | ● | See [README](README.md). Append-only: written at posting, frozen forever; `isActive` CHECK `true`. |

`UNIQUE (transactionId, name)`. **A row exists only when the amount does** — no
zero-filling (a deposit writes one row; an exempt coupon writes no `WITHHOLDING` row).

## The nine names

| Name | Formula | Written when |
|---|---|---|
| `QUANTITY` | = the `quantity` input, confirmed (entered-and-witnessed, like `ACCRUED`) | every lot event — the `ESTABLISH` / `RELIEVE_REALIZE` / `RESTATE` leg's referent and the basis for per-unit charge policies |
| `PRINCIPAL` | `quantity × price × quotationFactor` (factor on the instrument kind: 1 shares/funds, 0.01 percent-quoted bonds) | every priced lot event (`ESTABLISH` / `RELIEVE_REALIZE`) + every pure cash movement |
| `ACCRUED` | = the `accrued` input, confirmed — witnessed, not derived; the agreed figure wins | bond buy/sell, BRH early redemption, when > 0 |
| `COUNTER` | the settlement side of a conversion: **(Σ of the instrument-currency cash that settles) × fxRate** — `PRINCIPAL × fxRate` on a plain trade, `(PRINCIPAL + ACCRUED) × fxRate` on a bond buy with accrued, `INCOME × fxRate` on a converted income payment. Settlement cash instrument's currency; `fxRate` always quoted instrument → settlement. Gate: Σ converted cash legs = `COUNTER` | any transaction whose settlement cash instrument's currency ≠ the deal instrument's currency |
| `GROSS` | the income's full value: `nominal × rate × dayFraction` (coupons, from instrument params) · `qty held × dividendPerShare` (dividends, from the CA event) · recalculated-rate interest (TD early close) | every income event |
| `WITHHOLDING` | `GROSS × withholdingRate` — the instrument's fixed at-source rate (0.15 shares, 0.20 ordinary bonds), read at posting and frozen here; **row absent when the instrument is exempt** (`withholdingRate` NULL — OBL BRH) | taxed income |
| `INCOME` | `GROSS − WITHHOLDING` — what legally arrived; the income leg's referent | every income event |
| `BASE_INTEREST` | `nominal × rate × dayFraction` — the unprotected base | BRH coupon, **only when the index triggers** |
| `INDEX_SPREAD` | `BASE_INTEREST × (indexNow − indexRef) / indexRef` | same — with `GROSS = BASE_INTEREST + INDEX_SPREAD` as a gate check |

## Why frozen

Not because the arithmetic changes — `quantity × price` never will — but because the
**inputs** do: instrument parameters get corrected, tax rates get reformed. The row says
forever what actually applied that day. And because configuration binds **by name to
data**: a charge-policy row can point at `PRINCIPAL`; it cannot point at a Python
expression. One computation, N consumers (charges, accounting, statements, audit) —
instead of three re-implementations that drift.

**Who computes**: header-derived names (`QUANTITY`, `PRINCIPAL`, `ACCRUED`, `COUNTER`)
come from the generic engine; instrument-derived names (`GROSS`, `WITHHOLDING`,
`INCOME`, `BASE_INTEREST`, `INDEX_SPREAD`) come from the instrument type's registered
[calculator](../instruments/InstrumentCalculator.md) — then all are frozen here, the
single row every consumer binds to by name.

## The admission test

*Was this amount part of what was agreed and executed that day — unwaivable, with no
cash movement of its own?* Yes → a witness row. Otherwise:

- **Waivable, or settles with its own cash** → a **charge** (fees, commissions, TCA,
  penalties, exit fees) — auto-generated charge legs configured in the
  [Charges module](../charges/README.md), never a witness. Withholding is *not* a
  charge: it is the state's claim executed inside the payment — the client never
  possessed the gross, ProFin cannot waive it, its cash never moved separately.
- **Changes when a lifecycle moves** → a **read-time derivation**, never stored: market
  value, running accrual, "net after fees" (= `INCOME − Σ charge legs`, composed at
  render).

## Notes & rules

- Witness layer, **never truth**: positions, lots, and the disposal walk read legs only.
  If every row here vanished, no balance would change — you'd lose provenance and config
  bindings, not a financial fact.
- The market-making spread is **not** an amount — it materializes as NAOS realized P&L
  through the ordinary walk.
- Known tenth-name candidate, parked: third-party fees netted at source (Raymond James),
  pending confirmation of modalities.

## Clean model

```
TransactionAmount
  id             uuid    PK
  transactionId  FK Transaction
  name           enum (QUANTITY | PRINCIPAL | ACCRUED | COUNTER | GROSS |
                       WITHHOLDING | INCOME | BASE_INTEREST | INDEX_SPREAD)
  value          decimal(28,8)
  currency       char(3)?          -- NULL iff name = QUANTITY
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)
  unique (transactionId, name)
  -- append-only: isActive CHECK true; modifiedAt/By stay NULL
```
