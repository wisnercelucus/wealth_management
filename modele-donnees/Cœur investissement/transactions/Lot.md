# Lot

The **cost memory** — one acquisition batch, remembering what was paid from the day an
`ESTABLISH` leg creates it until the disposal walk fully relieves it. Lots live on the
**ledger side** (written atomically at posting), not in the Position module — which
*reads* them (unrealized P&L, holding periods) but owns none.

## Essential fields

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `portfolioId` | FK→Portfolio | ● | The walk's filter. |
| `instrumentId` | FK→Instrument | ● | The walk's filter. |
| `currency` | char(3) | ● | Snapshot of the instrument currency — same argument as the leg's: reference data is mutable; a lot's meaning is not. |
| `openTransactionId` | FK→Transaction | ● | Which `ESTABLISH` created it (for a child lot: the `RESTATE` transaction that restated it out of its parent). |
| `openDate` | date | ● | = the opening transaction's `tradeDate`. **The walk's `ORDER BY` (FIFO)** and the holding-period clock (NAOS exit fees read it through relief rows). Child lots copy the parent's `openDate`, never the restate transaction's date: the clock travels. |
| `postingSequence` | bigint | ● | Knowledge-order rank, assigned at creation. The FIFO tiebreak — `ORDER BY openDate ASC, postingSequence ASC` — and what makes FIFO-by-knowledge deterministic. |
| `qtyOriginal` | decimal(28,8) | ● | Frozen at creation. |
| `qtyRemaining` | decimal(28,8) | ● | Decremented by the walk. `position ≟ Σ qtyRemaining` is a standing verification. |
| `costOriginal` | decimal(28,8) | ● | **The exact `CARRY + PRINCIPAL` cash** of the opening transaction (or the declared `GROSS` basis for a stock dividend, or the cost share taken from the parent for a restate child), in `currency`. Frozen. Accrued (`OTHER`) and charges never enter — not by rule, but because the selector doesn't match them. |
| `costRemaining` | decimal(28,8) | ● | Mutated beside `qtyRemaining` by the same events. Cost as **totals** makes conservation literal: `Σ relief costs + costRemaining = costOriginal` — unit-cost arithmetic leaks rounding dust on multi-relief lots; totals cannot. |
| `fxAtOpen` | decimal(28,12) | ● | Instrument currency → portfolio reference currency. Sourcing: **the transaction's applied `fxRate`** when the opening transaction converted between the instrument currency and the book's reference currency; **the rates series as-of `openDate`** otherwise. `1` when instrument and book share a currency. One of the two witnesses the realized-FX split reads. |
| `parentLotId` | FK→Lot (self) | ○ | Restate lineage: the restate routine closes parent quantity into child lots carrying the parent's cost share, `openDate` and `fxAtOpen` — cost **and** holding clock travel. Lineage for audit. |
| + envelope | | ● | See [README](README.md). `isActive` CHECK `true` — one mutable pair of columns does not make a lot deactivatable; an exhausted lot (`qtyRemaining = 0`) is kept, never deleted. |

## Derived, never stored

`unitCost = costRemaining / qtyRemaining` (or `costOriginal / qtyOriginal` — the same
number until a future `ADJUST_BASIS` touches the lot) is **display arithmetic**. Position-level cost = Σ over open lots;
unrealized P&L = value − Σ(`costRemaining × fxAtOpen`) — all read-time.

## The ESTABLISH contract

At posting, the lot builder prices the lot from the transaction's `CARRY + PRINCIPAL`
cash leg **if present, else the stated basis** (`GROSS` for dividend-in-shares, the
`PRINCIPAL` witness for a cross-currency purchase — the paired cash leg is the common
case, not the definition). Cross-currency purchases are **native**: the cash leg carries
`COUNTER` in the settlement currency, the lot costs `PRINCIPAL` in the instrument's
currency, and `fxAtOpen` takes the applied `fxRate`. The two-step decomposition
(`FX_DEAL` + mono-currency trade under one `eventId`) remains available as operator
practice, never a requirement.

## The restate routine

A `RESTATE` leg posts (split, securities transfer) → the routine closes parent quantity
into **child lots**. It moves no cash and realizes nothing:

- **Child**: `qtyOriginal` = the restated quantity (× ratio for a split; = the moved
  quantity for a transfer), `costOriginal` = the cost share taken from the parent's
  `costRemaining`, `openDate` / `fxAtOpen` / `currency` copied from the parent,
  `parentLotId` set, fresh `postingSequence` (knowledge order).
- **Parent**: `qtyRemaining` / `costRemaining` decremented, the same two mutable columns
  the walk uses. `qtyOriginal` is never rewritten.
- **Conservation, per lot**: `Σ costRelieved + Σ children.costOriginal + costRemaining =
  costOriginal`. A restate never invents or loses a gourde, and because `openDate`
  travels, FIFO order and exit-fee ages survive any chain of transfers and splits.
- Reversing a `RESTATE` is legal only while its children are intact (mirror of the
  `ESTABLISH` rule); otherwise reject: "reverse the later events first."

## Notes & rules

- **Only `qtyRemaining` and `costRemaining` mutate** — and only inside the walk's, the
  restate routine's, a reversal's, or a future `ADJUST_BASIS`'s atomic block. Everything
  else is frozen at birth.
- **Cost-basis method is a walk strategy, never a data shape**
  (`Portfolio.costBasisMethod`): the MVP implements FIFO only — consume in
  `openDate ASC, postingSequence ASC` order. Because lots always store the full
  acquisition history, `AVERAGE_COST` (consume pro-rata across all open lots: the
  blended cost is exactly the position average), LIFO, or a custom order are future
  strategies added as code, with no schema change. `FUND` books are FIFO by business
  rule: exit fees key on the age of the *specific* units redeemed.
- Reversing an `ESTABLISH` is legal only while the lot is intact (`qtyRemaining =
  qtyOriginal`); otherwise reject — "reverse the sale first." Cascade order is
  explicit, never silent.
- `ADJUST_BASIS` (dormant): append-only adjustment rows + `costRemaining` decrement.
  Never rewrites cost history.

## Clean model

```
Lot
  id                 uuid    PK
  portfolioId        FK Portfolio
  instrumentId       FK Instrument
  currency           char(3)          -- snapshot
  openTransactionId  FK Transaction
  openDate           date             -- FIFO order + holding clock
  postingSequence    bigint           -- knowledge-order tiebreak
  qtyOriginal        decimal(28,8)
  qtyRemaining       decimal(28,8)    -- mutable (walk)
  costOriginal       decimal(28,8)    -- the PRINCIPAL cash, frozen
  costRemaining      decimal(28,8)    -- mutable (walk); Σ reliefs + this = costOriginal
  fxAtOpen           decimal(28,12)   -- applied fxRate if the opening converted; else rates series as-of openDate
  parentLotId        FK Lot? (self)   -- restate lineage (transfers, splits)
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)
  -- isActive CHECK true; exhausted lots kept forever
```
