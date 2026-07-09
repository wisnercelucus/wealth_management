# DRAFT — Position module (v2, all points settled in discussion)

Working draft, 2026-07-09. Every decision below was settled with the owner.
When approved, this becomes the final module docs (README + one file per table).

## What this module is

The ledger (legs + lots) is the truth. The Position module is a **cache** on top of
it, so a read does not sum ten years of legs every time. It stores **quantity only**.
Everything else — value, cost, accrual, P&L — is computed at read time. If the whole
module were dropped, no information would be lost: it can always be rebuilt from the
ledger.

The one rule that makes it safe: **the cache is updated, or marked stale, in the
same database save as the ledger write.** The system can never show an old number
as if it were fresh. (This makes the Axia 12-day desync bug structurally
impossible.)

## The two tables

```
Position
  id            uuid    PK
  portfolioId   FK Portfolio
  instrumentId  FK Instrument
  quantity      decimal(28,8)   -- trade-date basis: what the book holds economically
  isStale       bool            -- true = "do not trust me, recompute first"
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)
  unique (portfolioId, instrumentId)
```

```
PositionReconciliationFinding      -- one row per problem the nightly check finds
  id              uuid    PK
  portfolioId     FK Portfolio
  instrumentId    FK Instrument
  checkedAt       datetime
  cachedQuantity  decimal(28,8)   -- what the cache said
  ledgerQuantity  decimal(28,8)   -- what the legs say (the truth)
  resolvedAt      datetime?       -- when the recompute fixed it
  + envelope
```

Decided and closed:

- **One quantity, trade-date basis.** At ProFin, tradeDate = settleDate in practice
  (deal day = money day). A settled-cash view only differs during a settlement gap
  (e.g. a future Raymond James T+1 trade), and it is derivable from the legs when
  needed — the ledger stores both dates on every transaction forever, so nothing is
  lost by not storing it now. Adding a settled column later = one migration + one
  rebuild, no history problem.
- **No `lastPostingSequence`.** "What last touched this position" is a one-line
  query on the legs. Not stored.
- **No daily snapshot table** (Axia's `DailyPositions` rejected). No day-tick job.
- **`isStale` is a boolean** — every recompute is a full SUM anyway.
- **The findings table is real information, not a derivable**: it is the evidence
  of what went wrong, when, and when it was fixed. It answers "what changed and
  why" for the cache.
- No stored market value, cost, accrual, or P&L — ever.

## Workflows

### 1. Normal posting (the common case)

Inside the posting's database transaction, after the legs are written:

```
for each (portfolio, instrument) touched by the legs:
    quantity += Σ leg amounts
```

2–4 rows, cheap, synchronous, always consistent.

### 2. Backdated entry / reversal (the heavy case)

Same database transaction: set `isStale = true` on every affected row — cheap and
immediate. A worker recomputes later: `quantity = SUM(legs)`, one set-based SQL
statement. The period-close wall bounds how far back any recompute reaches.

### 3. Reads

- Fresh row → serve it.
- Stale row → compute the true value from the ledger on the spot, refresh the row,
  serve the fresh value. The cache heals itself; the worker is a backstop for rows
  nobody reads.
- **As-of a past date** ("what did the portfolio hold on March 31?") → sum the legs
  dated on or before that date. Nothing stored. (Which date per leg: the existing
  ledger rule — cash legs count on `settleDate`, all others on `tradeDate`.)
- **"What did the report we gave the client show?"** → that is the report archive
  (reporting module), which freezes a report's data when it is generated. A daily
  position table cannot answer this (the 4 pm report vs the 9 pm backdated entry),
  so none exists.

### 4. Valuation (read-time, never stored)

```
market value            = quantity × price(as-of) × quotationFactor     -- CLEAN value
value in reference ccy  = market value × FxRate(as-of)
```

- **Accrued interest is NOT inside market value.** It is its own figure — a
  dedicated column/report, computed live by the instrument's calculator. (Owner's
  ruling: accruals are shown separately, not mixed into value.)
- The `POSITION_VALUE` base of scheduled charge policies reads the **clean** value.
- Prices and rates come from market-data with the as-of rule (most recent row on or
  before the date). Consumers: portfolio views, AUM (`bookKind = CLIENT` ∩
  asset-class eligibility), statements, charge policies.

### 5. Unrealized P&L (read-time, never stored)

Computed over the position's **open lots**, against `costRemaining` — the cost of
what is *still held*. (Bought 100 for 1,000, sold 40: the profit on the 40 is
already frozen in LotRelief; the 60 still held cost 600 = `costRemaining`.
Unrealized = today's value of the 60 − 600.)

```
unrealized price P&L = market value − Σ costRemaining              (instrument ccy)
unrealized FX P&L    = Σ costRemaining × (fxNow − fxAtOpen)        (reference ccy)
```

Same price/FX split as realized P&L in LotRelief. Mono-currency books: rates are 1,
FX part is 0, no special case.

### 6. Performance — MWR, net of fees

One method in the MVP: **money-weighted return = gain ÷ average capital**
(time-weighted average capital as the base). It answers the client's question:
"how much did my money earn?" TWR is not built.

- External flows = cash legs of transactions with
  `cashFlowCategory = EXTERNAL_IN / EXTERNAL_OUT`.
- **Charge legs (`chargeNameId` set) are never external flows** — they stay inside
  the gain, so the return is **after fees**. A withdrawal is a flow; its fee is a
  cost.

### 7. Reconciliation = recovery (one machinery)

Nightly: recompute every position changed that day, plus a random sample of
unchanged rows (silent-drift detection). Compare with `Σ legs` and with
`Σ Lot.qtyRemaining`. Mismatch → write a **PositionReconciliationFinding** row,
mark stale, recompute, alert. A full rebuild is the same machinery run on every
row. There is no separate repair tool.

## What lives elsewhere (boundaries)

- **Lots and the disposal walk** → the transactions domain, inside the atomic
  posting block (an oversell must reject the transaction, so the walk can never be
  deferred work). This module only reads lots.
- **Running accrual** → instrument calculators, at read time. Three-way split:
  transacted accrual = witness on the ledger · running accrual = read-time ·
  as-filed accrual = report archive.
- **Report archive** (as-filed data snapshots: BRH submissions first, client
  statements next) → the reporting module.
- **NAV immutability** (a published NAV freezes when the first unit transacts
  against it) → the fonds module. Recorded here so it is not lost.

## Impacts on other docs when finalized

- transactions README: "the Position module is notified after posting" → becomes
  "position rows are updated or marked stale **in the same database transaction**;
  the heavy recompute is deferred." (Owner ratified.)
- FOLLOW-UP entry with the decision batch.
