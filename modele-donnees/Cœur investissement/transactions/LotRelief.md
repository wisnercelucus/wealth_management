# LotRelief

The **realized P&L of record** — one row per lot consumed by a disposal. Written by the
walk inside the posting's atomic block, frozen forever: the two-component split (price
vs FX) lands here and is summed by every realized-P&L report, never recomputed.

## Essential fields

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `lotId` | FK→Lot | ● | The lot consumed — always a specific acquisition batch. A future average-cost strategy consumes pro-rata across all open lots: more rows, same mechanics. |
| `closeTransactionId` | FK→Transaction | ● | The disposal (`RELIEVE_REALIZE`) that ran the walk. |
| `qtyRelieved` | decimal(28,8) | ● | How much of the lot this row consumed. Σ per closing transaction = the `RELIEVE_REALIZE` leg's quantity — a standing invariant. |
| `proceedsAmount` | decimal(28,8) | ● | This row's share of the disposal's `PRINCIPAL` cash, allocated pro-rata by quantity, **final row absorbs the remainder** — the dust-free allocation. Instrument currency. When the disposal's settlement converted (cash legs in another currency), the walk takes proceeds from the `PRINCIPAL` **witness** — the mirror of the lot builder's cross-currency rule. |
| `costRelieved` | decimal(28,8) | ● | Taken from `Lot.costRemaining` pro-rata, final take absorbs the remainder. `Σ costRelieved + costRemaining = costOriginal` holds exactly — "never invents or loses a gourde," arithmetically. |
| `fxAtClose` | decimal(28,12) | ● | Instrument currency → reference currency. Sourcing: **the disposal's applied `fxRate`** when it converted between the instrument currency and the reference currency; **the rates series as-of the disposal's `tradeDate`** otherwise. `1` on mono-currency books. |
| `realizedPricePnl` | decimal(28,8) | ● | `proceedsAmount − costRelieved` — **instrument currency, frozen.** |
| `realizedFxPnl` | decimal(28,8) | ● | `costRelieved × (fxAtClose − lot.fxAtOpen)` — **reference currency, frozen.** Mono-currency books: both rates are 1, this is 0 — no special case. |
| + envelope | | ● | See [README](README.md). Append-only: `isActive` CHECK `true`; `modifiedAt/By` stay `NULL`. |

## The walk (summary — full trace in the workflow doc)

A `RELIEVE_REALIZE` leg posts → select the position's open lots
`ORDER BY openDate ASC, postingSequence ASC` (FIFO-by-knowledge — the MVP's only
strategy; a future `AVERAGE_COST` walks the same lots pro-rata), **`FOR UPDATE`** under
per-(portfolio, instrument) serialization →
consume in order, one relief row per lot touched, each carrying **its own lot's**
witnesses → oversell (quantity left after the last lot) rejects the whole transaction.
Only `qtyRemaining`/`costRemaining` mutate; everything written here is born frozen.

## Standing invariants (cheap, always-on tests)

- `realizedPricePnl × fxAtClose + realizedFxPnl` = total reference-currency P&L per
  row — the identity holds exactly and is the cheapest FX-sourcing bug detector.
- Σ `qtyRelieved` per closing transaction = the disposal leg's quantity.
- Σ `costRelieved` + Σ child-lot `costOriginal` + `Lot.costRemaining` =
  `Lot.costOriginal`, per lot (children come from the restate routine: transfers, splits).
- position ≟ Σ `qtyRemaining` per (portfolio, instrument).

## Notes & rules

- **Reversal is offsetting, never deletion**: reversing a disposal writes counter-relief
  rows (negative quantities and amounts) under the reversing transaction and restores
  `qtyRemaining`/`costRemaining`. Append-only applies to reliefs too.
- **Holding periods read through this table**: the NAOS exit-fee generator reads each
  relief row's `lot.openDate` — the walk answers the fee question as a by-product (the
  same walk, two reads). This is why the Charges module's read contract includes
  LotRelief.
- **Backdating (FIFO-by-knowledge)**: a late-entered older lot waits its turn behind
  walks that already ran; relief rows are never disturbed. P&L is reordered between
  disposals within the open month, never invented or lost. The entry gate stamps such
  collisions — loud, lawful, and rare by operational restriction.
- The Accounting module reads this table to journal realized P&L — it cannot be
  projected from types + witnesses + charge lines alone.

## Clean model

```
LotRelief
  id                  uuid    PK
  lotId               FK Lot
  closeTransactionId  FK Transaction
  qtyRelieved         decimal(28,8)
  proceedsAmount      decimal(28,8)    -- pro-rata, final row absorbs remainder
  costRelieved        decimal(28,8)    -- pro-rata from costRemaining, same rule
  fxAtClose           decimal(28,12)   -- applied rate if the disposal converted; else series as-of disposal tradeDate
  realizedPricePnl    decimal(28,8)    -- instrument ccy, frozen
  realizedFxPnl       decimal(28,8)    -- reference ccy, frozen
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)
  -- append-only: isActive CHECK true; reversal = offsetting rows
```
