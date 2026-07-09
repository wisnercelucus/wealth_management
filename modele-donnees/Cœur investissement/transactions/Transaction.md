# Transaction

The **event header** — one row per economic event. Carries what the operator (or emitter)
supplied and the event's lifecycle; everything computed lives in
[TransactionAmount](TransactionAmount.md), everything that moved in
[TransactionLeg](TransactionLeg.md). *The transaction is the event; the legs are its
addresses.*

## Essential fields

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `transactionTypeId` | FK→TransactionType | ● | The catalogue type. Frozen behavior — see [TransactionType](TransactionType.md) immutability. |
| `portfolioId` | FK→Portfolio | ● | **The deal's own book.** What contract modes `TRANSACTION` and `TRANSACTION_CASH` resolve against (the latter via the portfolio's `cashSettlementPortfolioId`). |
| `instrumentId` | FK→Instrument | ○ | **The deal's instrument.** What eligibility validates at entry and `instrumentMode = TRANSACTION` reads. Required **iff** any contract line of the type uses that mode or resolves its portfolio via `TRANSACTION_INSTRUMENT_ISSUER`. For pure cash types (deposit, withdrawal, cash transfer) the deal's instrument **is** the picked cash instrument. One per transaction — the settlement cash side has its own slot (`settlementCashInstrumentId`); a future merger CA gets its second instrument from the event, not the operator. |
| `bankAccountId` | FK→BankAccount | ○ | Operator's bank-account pick. Required **iff** a contract line uses `TRANSACTION_BANK`; resolves to the unique house book carrying that account. |
| `counterpartyPortfolioId` | FK→Portfolio | ○ | The other party's book (client transfer; future inter-client trade). Required **iff** a contract line uses `TRANSACTION_COUNTERPARTY` or `TRANSACTION_COUNTERPARTY_CASH`. |
| `quantity` | decimal(28,8) | ○ | Input — the deal's size in the instrument's unit; **money is the quantity for cash**. Magnitude only: `CHECK (quantity IS NULL OR quantity > 0)` — direction lives on contract lines. |
| `price` | decimal(28,12) | ○ | Input — clean price / NAV / rate; `1` for cash; `100` = par at maturity. The instrument kind's `quotationFactor` (1 shares/funds, 0.01 percent-quoted bonds) keeps the executor free of kind conditions. |
| `accrued` | decimal(28,8) | ○ | Input — transacted accrued, as agreed. The witnessed figure wins over recomputation; disputes travel through reversal. |
| `fxRate` | decimal(28,12) | ○ | Input — the **applied conversion rate** of any transaction whose settlement cash instrument's currency differs from the deal instrument's (an FX buy/sell is simply this with a cash instrument as the deal's instrument). Always quoted instrument → settlement. Pre-filled from the rates series as-of `tradeDate`, operator-editable; what was applied is frozen (witnesses, legs, lot). The gate records the deviation from the series default on every override. Required **iff** the transaction embeds a conversion; absent otherwise (same currency ⇒ no conversion exists, rate is 1). |
| `settlementCashInstrumentId` | FK→Instrument | ○ | **The cash instrument the money leg moves through** (CASH kind, gate-validated). Default: the settlement book's declared cash instrument (`Portfolio.cashInstrumentId`). When its currency differs from the deal instrument's, the trade is cross-currency: `fxRate` becomes required and the cash leg is fed by `COUNTER`. Pure cash types don't use it — there the cash instrument *is* the deal's `instrumentId`. |
| `tradeDate` | date | ● | **The economic date.** The period wall derives from it (must fall in the open month); position views filter on it; the accounting period is never a stored column. |
| `settleDate` | date | ● | Settlement. `CHECK (settleDate >= tradeDate)`. CASH-kind legs take effect on settleDate; all others on tradeDate — a derived rule, never a stored column. |
| `status` | enum | ● | `PENDING \| POSTED \| REJECTED`. A `PENDING` row is a **proposal** — header only, no amounts, no legs, and **not a hold** on balances. Truth materializes at approval. Immutable once `POSTED`. |
| `postedAt` | datetime | ○ | Set inside the atomic posting block; `NULL` until then. The PENDING→POSTED flip is otherwise untimestamped (append-only keeps `modifiedAt` null). |
| `postingSequence` | bigint | ○ | Monotonic rank assigned at posting. **The knowledge order** — what FIFO-by-knowledge and backdating audit actually sort on (uuid PKs carry no order). |
| `reversesTransactionId` | FK→Transaction (self) | ○ | Reversal = a **new** transaction pointing at the original — open month only. Partial `UNIQUE` (a transaction is reversed at most once); reversing a reversal is forbidden — re-post the original economics as a new transaction. |
| `eventId` | uuid | ○ | Groups the transactions of one multi-transaction event: the market-making pair, a CA batch, the FX-deal + trade pair of a cross-currency purchase. |
| `sourceSystem` | string(50) | ○ | Which emitter wrote this (CA, scheduler, migration…). Partial `UNIQUE (sourceSystem, externalId)` — the idempotency key; emitters must supply both. |
| `narrative` | text | ○ | Statement text — generated from the type's `narrativePattern` at posting or entered. Carries correction stories for closed-month fixes. |
| + envelope | | ● | See [README](README.md). `createdBy` = the maker; checker identity lives in the Approval module, linked. `externalRef` carries business references (wire ref, RJ confirmation). |

## Deliberately absent

- **Trade currency** — derivable from the legs' instruments.
- **Amount** — `quantity × price`; money *is* quantity for cash.
- **Period id** — derived from `tradeDate`; closed months are immutable and corrections
  are forward-dated, so nothing can ever land in a month it isn't dated in.
- **Approval fields** (`approvedAt/By`) — the Approval module's ledger, linked not
  duplicated. `postedAt` is the ledger-side timestamp.
- **Any balance, market value, or accounting field.**

## Notes & rules

- **Five inputs is the complete deal-terms surface.** Every MVP type resolves from
  `quantity / price / accrued / fxRate / settlementCashInstrumentId` plus the three
  address columns. A new input column requires a genuinely new *resolution source*
  (rare, deliberate — `settlementCashInstrumentId` earned its place as the cash side's
  pick); a new leg never does — that's a contract line.
- **Address columns are required-iff**: each is mandatory exactly when the type's
  contract contains the mode that reads it, enforced at the entry gate.
- Rounding: witnesses are rounded once at computation to currency minor units; gate
  validations compare post-rounding.

## Clean model

```
Transaction
  id                       uuid      PK
  transactionTypeId        FK TransactionType
  portfolioId              FK Portfolio
  instrumentId             FK Instrument?        -- required iff instrumentMode TRANSACTION or portfolioMode TRANSACTION_INSTRUMENT_ISSUER in contract
  bankAccountId            FK BankAccount?       -- required iff TRANSACTION_BANK in contract
  counterpartyPortfolioId  FK Portfolio?         -- required iff a _COUNTERPARTY mode in contract
  quantity                 decimal(28,8)?        -- > 0 when present
  price                    decimal(28,12)?
  accrued                  decimal(28,8)?
  fxRate                   decimal(28,12)?       -- applied conversion rate; required iff conversion embedded; series default, operator-editable
  settlementCashInstrumentId FK Instrument?      -- cash-side pick (CASH kind); default = the settlement book's cashInstrumentId
  tradeDate                date                  -- the economic date; period derives from it
  settleDate               date                  -- >= tradeDate
  status                   enum (PENDING | POSTED | REJECTED)
  postedAt                 datetime?             -- set in the atomic block
  postingSequence          bigint?               -- knowledge order
  reversesTransactionId    FK Transaction? (self) -- partial unique; no reversing a reversal
  eventId                  uuid?
  sourceSystem             string(50)?           -- unique (sourceSystem, externalId) where not null
  narrative                text?
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)
  -- POSTED rows: immutable; isActive CHECK true; modifiedAt/By stay NULL
```
