# Follow-up — decisions and pending corrections

Working log opened after the transaction-domain verification of 2026-07-03. Each item is
either a decision taken (with the doc edits it implies) or a decision still open.

## Decisions taken and applied (2026-07-03 → 2026-07-08)

### Market-data module created (2026-07-08)
The deferred component, done in one pass — two observation tables, no computation:
**InstrumentPrice** (one price per instrument per date; securities, fund NAVs, and
index/benchmark values alike — the BRH index series lives here) and **FxRate** (one
rate per currency pair per date, in the pair's base → quote convention; consumers
derive direction). "As-of date D" = the most recent row on or before D. Corrections
edit rows freely: the ledger never re-reads the series — every applied value is
frozen at posting (header fxRate, witnesses, fxAtOpen/fxAtClose, legs). Pre-fill with
no row for the date leaves the field empty and the operator supplies the value (gate
deviation stamping unchanged).
Applied: market-data/ created (README, InstrumentPrice.md, FxRate.md); the
deferred-by-decision entry below is closed.

### Charges module finalized: charges are legs, configuration is the module (2026-07-08)
Design session closing DRAFT v1–v3 (taxation/DRAFT-charges-module.md, deleted —
superseded by the new charges/ module docs). The decisions:

- **No instance tables.** TransactionChargeAmount (and any ChargeLine) dropped: a
  charge is always exactly its movement, so a stored charge figure would be a banned
  derivable. A posted charge lives in the ledger as TransactionLeg rows stamped with a
  new nullable `chargeNameId` FK (charge-analysis filter and is-charge boolean in one
  column; CHECK exclusive with `amountName`).
- **Charge legs are automatic, never authored.** One leg per charge value: − the
  settlement cash on the client side, stamped chargeNameId. No destination column and
  no house-side leg (destinationPortfolioId considered, then dropped by the owner):
  the receiving side (commission revenue, TCA payable) is the Accounting module's
  product at the GL, bound by charge name. No Σ=0 requirement — the ledger is
  per-book truth, not double-entry. Charge legs are CARRY / principalFlag NULL:
  charges never enter cost basis.
- **Manual and automatic are strictly separate and additive; no pre-fill.** Declared
  fields (TransactionTypeCharge) are the operator's deliberate charges and ride the
  entry transaction; policies are tariffs. Overlap is legitimate (tariff + deliberate
  extra); optional read-only UI notice of applicable tariffs. Declaring/removing a
  charge on a posted type never needs a new type code (charges never touch contract
  lines).
- **Where a tariff lands follows its trigger.** TRANSACTION-triggered policies run
  inside the posting block of every matching transaction and land on it; SCHEDULE
  policies post their own charge transaction per book and period (type with no
  contract lines; provenance/idempotency via sourceSystem = CHARGE_ENGINE +
  externalId run key). No chargePolicyId column anywhere — nothing computes from it;
  validity dates reconstruct what ruled.
- **Tax follows the service**: ChargeName.taxChargeNameId + taxRate derive the tax
  (COMMISSION → TCA) in the same transaction; no tax on tax; tax names never declared
  nor output. Absence of the parent is the waiver of both.
- **BRH maturity commission is on the fixed gain only** — the index spread is
  excluded, even though the client is paid GROSS when the index triggers. Witness
  semantics unchanged (owner: the nine names capture every needed amount); the
  commission policy's base is picked at policy authoring against the transactions the
  maturity flow actually emits.
- **Charge-leg resolution stated; FUND books declare a cash instrument** (audit
  closure, same day). A charge leg resolves like every cash leg: − on the settlement
  book of the transaction's portfolio (the book itself when it self-settles), in that
  book's declared `cashInstrumentId`. `Portfolio.cashInstrumentId` is now also
  required on `FUND` books (owner: fund books are normal portfolios, their own
  settlement cash) — closes the scheduled-fee-on-NAOS resolution gap. Charge inputs
  need no pending storage: nothing materializes before approval; entered values ride
  the entry/approval payload.

Applied: charges/ module created (README, ChargeName, ChargePolicy,
ChargePolicyCondition, ChargePolicyEffect); transactions/TransactionTypeCharge.md
(the type's declaration table lives with the type, per the
TransactionTypeEligibility precedent); TransactionLeg.md
(chargeNameId field, auto-generation note, charge-analysis index, clean model);
TransactionAmount.md (admission test now routes to charge legs; read-time
"net after fees" wording); transactions README (boundary quote, where-things-live);
TransactionType.md (declared-charges note).

### InstrumentGroup added (2026-07-08)
New grouping entity in the instrument cluster, mirroring PartyGroup / PortfolioGroup
(group + explicit membership): the instrument-side scope handle for charge and taxation
policies (CDC "groupe instrument"). Distinct from kind (what it is) and asset class
(how it behaves). Membership is explicit, so a rule that must catch every instrument of
a kind automatically scopes on the kind instead.
Applied: instruments/InstrumentGroup.md created; instruments README documents list.

### Fund books self-settle; subscription is an ordinary issuer transaction (2026-07-07)
Owner's clarifications. A fund (NAOS) is a client too; everything the fund does happens
in its single FUND book, so FUND and house books **self-settle**:
`cashSettlementPortfolioId` stays NULL and `TRANSACTION_CASH` resolves to the deal book
itself ("NULL = cash account" is now scoped to client books). A fund subscription is an
ordinary issuer transaction from a ProFin client: the fund instrument's
`issuerPortfolioId` points at the PF Trustee issuer book, itself bank-linked. Same
pass: the CASH instrument mode now states that `TRANSACTION_INSTRUMENT_ISSUER` legs
take the header settlement cash like trade legs, and a bank-linked house book's
`cashInstrumentId` must match its bank account's currency (gate-validated).
Applied: Portfolio.md (cashSettlementPortfolioId, cashInstrumentId),
TransactionTypeLeg.md (TRANSACTION_CASH resolution, CASH mode).

### Review closure batch (2026-07-07)
Small fixes from the transaction/instrument cross-review: TransactionTypeLeg
isConditional example no longer cites a withholding-fed leg (none exists; tax is a
witness only); instruments README out-of-scope line now excepts the instrument's own
withholdingRate; INDEX stated as never eligible (InstrumentType row wording and a
TransactionTypeEligibility rule: creating an eligibility row for INDEX is rejected);
quotationFactor typed decimal(10,6) and withholdingRate decimal(5,4); transactions
README ERD gains the Transaction→Instrument, Lot→Instrument and
Eligibility→InstrumentType edges; InstrumentCalculator amountName lists drop the
trailing ellipsis (the nine-name set is closed).

### TRANSACTION_INSTRUMENT_ISSUER: seventh portfolio mode (2026-07-07)
New follower mode: reads the existing header `instrumentId` and walks the new
`Instrument.issuerPortfolioId` link to the issuer-side book the instrument settles
into. A per-party unique issuer book was rejected: an issuer can run several issuer
books, one per product family (CIC émetteur / CIC émetteur obligations), so the
instrument declares its own. `issuerPortfolioId` is optional on Instrument and
gate-required at posting when the type's contract uses the mode (NULL → reject at
entry). The issuer leg carries the settlement cash instrument (instrumentMode CASH):
the same money that leaves the client is tracked into the issuer book. The portfolio
mode set closes at seven.
Applied: Instrument.md (issuerPortfolioId), TransactionTypeLeg.md (mode table, counts,
follower example, clean model), Transaction.md (instrumentId required-iff, clean model).

### Terminology sweep: kind vs class (2026-07-07)
The instrument cluster reserves *kind* for InstrumentType and *class* for AssetClass;
the transaction docs said "instrument class" where they meant the kind. Swept:
Transaction.md (quotationFactor wording, CASH-kind settle-date rule), TransactionAmount.md
(PRINCIPAL factor), TransactionLeg.md (effective-date rule), TransactionTypeEligibility.md
(intro, field, isDefault, OBL BRH note, validation note), transactions README (ERD label).
No model change.

### Cash instrument is declared per book (2026-07-07)
Several cash instruments per currency are allowed on purpose (special-purpose cash),
so "the cash instrument in currency X" was ambiguous for the automatic resolutions
(settlement default on trades, TRANSACTION_BANK legs, FX shape). Fix: new
`Portfolio.cashInstrumentId` (FK Instrument, CASH kind, gate-validated). Required when
`portfolioType = CASH` and on house books carrying a `bankAccountId`; NULL otherwise.
Resolutions now read the book: the settlement default on a trade is the settlement
book's declared cash instrument; TRANSACTION_BANK legs use the bank house book's.
Cross-currency is unchanged: settlement instrument currency ≠ deal instrument currency
⇒ `fxRate` required. Pure cash types are untouched (operator picks the cash instrument
as the deal's instrument).
Applied: Portfolio.md (field, clean model), Transaction.md (settlement default),
TransactionTypeLeg.md (CASH-mode resolution), InstrumentType.md (CASH row wording).

### Type authoring is one screen; coherence re-checked on every change (2026-07-07)
A transaction type cannot be created without selecting its eligible instrument types on
the same authoring screen as `declaredAmounts`: the coherence check (required amount →
computable by every eligible kind; optional → by at least one) runs at save, so no type
ever exists without validated eligibility. Later changes (including adding a kind to an
existing type) go through the same screen and re-run the check. Deactivating an
InstrumentCalculator row is rejected while an active type still declares that amount
and is eligible for that kind.
Applied: TransactionTypeEligibility.md (authoring note), TransactionType.md
(declaredAmounts wording), InstrumentCalculator.md (re-check and deactivation rules).

### InstrumentCalculator.md trimmed to logic level (2026-07-07)
Implementation details are not settled yet, so the doc keeps the logic only: the code
samples (annotation decorator, collision-guard snippet, framework import and hot-reload
notes) and the functionKey seed table are removed. The registry is now stated as five
guarantees (code declares itself; UI selects, never types; startup validation; collision
failure at boot; built once per process, read-only) and the seeding logic is prose
(sparse rows per type; WITHHOLDING and INCOME as shared functions; OBL BRH exempt).
Concrete seed rows are decided with each instrument type part. In the same pass the
authoring-time coherence rule was aligned: a required amount needs a calculator row on
every eligible instrument type; an optional amount needs one on at least one (absent on
the others), which matches the OBL BRH exemption sentence.

### Change order 2026-07-07: computation moves to the instrument layer (applied)
Supersedes the DATA | CODE model. Transaction types are now **pure declaration**:
`behaviorMode` and `postingFunctionKey` removed from TransactionType; `declaredAmounts`
carried by every type; `isSystem` means only seeded / undeletable / read-only lines;
`isConditional` loses its CODE check (a line is emitted only when its amount applies).
Leg structure is always data; stages 2–4 of posting (legs → lot effects → position) are
one generic engine for every type. Amount computation routes by the amount **name**:
header-derived names (QUANTITY, PRINCIPAL, ACCRUED, COUNTER) via the generic engine;
instrument-derived names (GROSS, WITHHOLDING, INCOME, BASE_INTEREST, INDEX_SPREAD) via
the new **InstrumentCalculator** registry in the instrument cluster — rows map
(instrumentTypeId, amountName) → a code-registered `functionKey` (annotation-built
registry, dropdown-only selection, startup validation, boot-time collision guard).
Authoring-time coherence: a type may declare an instrument-derived amount only if every
eligible instrument type has a calculator row for it.

Modeling choice noted: WITHHOLDING and INCOME are per-type registry rows pointing at
shared functions (`withholding.compute`, `income.computeNet`), keeping the coherence
rule uniform; OBL BRH carries no WITHHOLDING row (exempt).

Applied: TransactionType.md (rewritten sections), TransactionTypeLeg.md (intro,
amountName, isConditional, examples untagged), TransactionAmount.md (who-computes note),
transactions README (posting pipeline, entity bullets), new
instruments/InstrumentCalculator.md, instruments README and InstrumentType.md
cross-links.

### One house book per bank account
Rule confirmed by the owner: a bank account picked by a portfolio cannot be picked by
another. Applied to Portfolio.md: partial `UNIQUE (bankAccountId)` where not null, noted
on the field and in the clean model. The ledger's `TRANSACTION_BANK` mode therefore
always resolves a bank account to a single house book.

### Fund books get their own bookKind
`bookKind` gains `FUND` (an in-house fund's own book, e.g. NAOS). Only `CLIENT` counts
toward AUM, so fund books are excluded and the double count (client fund units + the
fund's own holdings) is resolved. `FUND` books require FIFO (exit fees read the age of
the specific units redeemed). Applied to Portfolio.md: enum, AUM note, costBasisMethod
note, portfolioType wording, clean model.

### RESTATE lot effect (split and transfer mechanics)
Accepted and applied. Fifth `lotEffect = RESTATE`: quantity or location changes at
constant cost and constant holding clock; no cash, no P&L. A split is quantity-delta
legs; a securities transfer is − at source / + at destination. The restate routine
closes parent quantity into child lots (cost share, `openDate`, `fxAtOpen` travel,
`parentLotId` lineage); the parent only decrements `qtyRemaining` / `costRemaining`.
Per-lot conservation: `Σ costRelieved + Σ children.costOriginal + costRemaining =
costOriginal`.

Applied to: TransactionLeg.md (lotEffect list, CARRY cleaned up, principalFlag note,
clean model), TransactionTypeLeg.md (enum, SECURITIES_TRANSFER worked example, clean
model), Lot.md (restate routine section, parentLotId / openDate / openTransactionId /
costOriginal rows, mutation rule), LotRelief.md (conservation invariant),
transactions README (executor sentence), TransactionAmount.md (QUANTITY and PRINCIPAL
"written when", split example replaced). CDC coverage now closes at 100%; verification
findings D-1 and D-7 are resolved, and D-6 (the "five modes" count) was fixed in passing.

### Cost basis: FIFO only in the MVP, FIFO the default
Decided 2026-07-03. Reasons: NAOS exit fees require FIFO lot ages; no Haitian tax or BRH
rule prescribes a method (BRH is prudential, investment income is taxed by withholding at
source); operations and clients do not reconcile on realized gains; multi-buy partial
sells are rare, and realized P&L is declared FIFO where they occur. The backbone stays
method-flexible by construction: lots keep the full acquisition history forever, so
`AVERAGE_COST` (pro-rata walk), LIFO or a custom order are future walk strategies, code
only, no schema change. Switching a portfolio's method applies to future disposals only
(posted reliefs are frozen truth).

Applied: Portfolio.md (enum now `FIFO | AVERAGE_COST`, default FIFO, LIFO dropped,
cost-basis note rewritten), Lot.md (`lotKind ACQUISITION | POOL` removed — every lot is
now frozen with no carve-out; walk-strategy note), LotRelief.md (pool references replaced
by the strategy note). Optional verification noted: one-line confirmation from the
accountant that the DGI has no methodology rule on plus-values mobilières.

### FX and multi-currency: native cross-currency transactions, unified rate sourcing
Decided 2026-07-03. Two rates are distinct: the instrument price (the amount needed in
the instrument's currency) and the FX rate (the money taken from the cash book to
produce it). Transactions are multi-currency capable by default; the mono-currency case
is the same code path with rate 1.

- New header input `settlementCashInstrumentId` (renamed from settlementCurrencyId,
  same day): the cash instrument the money leg moves through. Default = the cash
  instrument in the deal instrument's currency (no conversion, no rate, no `COUNTER`
  row). Picking one in another currency makes the trade cross-currency. Pure cash types
  (deposit, withdrawal, cash transfer) don't use it: they pick the cash instrument as
  the deal's own instrument (header `instrumentId`, mode TRANSACTION), which also lets
  eligibility validate them like any other type.
- `fxRate` redefined: the applied conversion rate of any transaction embedding a
  conversion (cross-currency trade, FX deal). Pre-filled from the rates series as-of
  tradeDate, operator-editable, frozen at posting: what the operator chose is what
  stays. The gate records the deviation from the series default on every override
  (approval thresholds configurable in the Approval module).
- Both amounts are recorded: `PRINCIPAL` (instrument currency) and
  `COUNTER = PRINCIPAL × fxRate` (settlement currency) as witnesses; the cash leg
  carries the `COUNTER` money; the lot costs `PRINCIPAL` with `fxAtOpen` = the applied
  rate.
- Witness sourcing rule: `fxAtOpen` / `fxAtClose` take the transaction's applied rate
  when the transaction converted between the instrument currency and the book's
  reference currency, the rates series as-of the trade date otherwise, and are 1 on
  mono-currency books.
- The two-step decomposition (FX deal + mono-currency trade under one eventId) becomes
  optional operator practice, never a requirement.

Applied: Transaction.md (fxRate, settlementCashInstrumentId, instrumentId note,
five-input note), TransactionAmount.md (COUNTER scope), TransactionTypeLeg.md (CASH-mode
resolution, CASH_DEPOSIT reworked and CASH_TRANSFER added as worked examples),
TransactionTypeEligibility.md (cash / provision types validate via their header
instrument), Lot.md (fxAtOpen sourcing, ESTABLISH contract), LotRelief.md (fxAtClose
sourcing). Resolves parked finding D-3 fully and most of D-4. The market-data input
module (prices per instrument per date, FX rates per currency pair per date) is
confirmed as a component.

### TRANSACTION_COUNTERPARTY_BANK dropped: the portfolio-mode set closes at six
Decided 2026-07-03 (owner's catch). Under the one-book-per-bank-account rule the mode
was a round trip: it resolved to the counterparty book itself (house case) or rejected
(client case), so it never answered "which book" differently from
TRANSACTION_COUNTERPARTY, and no MVP operation used it. The generative rule is now
stated as: each source column × its available links, pruned of degenerate combinations.
If a type ever needs the guarantee "counterparty must be a bank-backed house book,"
that is gate validation, not a resolution mode.

Applied: TransactionTypeLeg.md (mode table, growth rules, clean model, counts),
Transaction.md (counterpartyPortfolioId required-iff now names both _COUNTERPARTY
modes, which closes finding D-2).

### FX lands as an ordinary cross-currency cash trade; COUNTER joins the universal set
Decided 2026-07-03. An FX deal is the cross-currency purchase / sale of a cash
instrument: header instrument = the foreign cash (e.g. Cash-USD), quantity = the foreign
amount, price = 1, settlement slot = the local cash, fxRate = the conventional quote
(always instrument → settlement). Witnesses PRINCIPAL + COUNTER; two CARRY legs on the
client's cash book; no lots. ProFin's side of the double rate is the same type posted on
house books (TRANSACTION + TRANSACTION_BANK) at the bank rate, grouped with the client
deal under one eventId; the FX book's net position falls out of the house books' cash
positions by currency.

COUNTER is promoted into the DATA universal set (QUANTITY, PRINCIPAL, ACCRUED, COUNTER:
the amounts computable from header inputs alone), so FX_BUY / FX_SELL are DATA types,
admin-authorable. The CODE triggers are restated as: instrument parameters, withholding
data, event data, or conditional legs.

Applied: TransactionType.md (universal set, boundary sentence), TransactionAmount.md
(COUNTER unified to PRINCIPAL × fxRate with the quoting convention),
TransactionTypeLeg.md (amountName restriction, FX_BUY worked example),
TransactionTypeEligibility.md (every type validates; the FX exception removed),
Transaction.md (fxRate wording). Resolves finding D-4 fully.

### Stress test (7-witness event) and the COUNTER generalization (applied 2026-07-03)
A BRH maturity with triggered index, settled into Cash-USD, was pushed through the model
as a stress test. The structure held (nine amount names suffice, legs and lot mechanics
compose); three definition fixes came out and were applied:

- `COUNTER` generalized: the settlement side of a conversion is **(Σ of the
  instrument-currency cash that settles) × fxRate**, not `PRINCIPAL × fxRate` (which
  misses the accrued on a bond buy and cannot be computed on a converted income
  payment). New gate identity: Σ converted cash legs = `COUNTER`.
  (TransactionAmount.md; TransactionType.md universal-set formula.)
- Converted settlements on legs: each cash leg carries its source amount × the header
  `fxRate`; no per-leg converted witnesses, everything reconstructible.
  (TransactionLeg.md.)
- The disposal walk mirrors the lot builder: when the settlement converted, proceeds
  come from the `PRINCIPAL` witness, since no instrument-currency cash leg exists.
  (LotRelief.md.)

Convention recorded (owner): **maturity-driven events emit two transactions**, one for
the principal redemption and one for the interest, under one `eventId` (BRH maturity +
BRH interest; ordinary-bond redemption + final coupon). Emitted by the Maturities
module through the ordinary posting gate.

### Withholding is a fixed per-instrument rate (applied 2026-07-03)
15% for shares (dividends), 20% for ordinary bonds (coupons), OBL BRH exempt. The rate
is entered on the instrument; there is no client × instrument tax matrix (the CDC
wording on this point is superseded). Applied: `withholdingRate decimal?` on
Instrument.md (NULL = exempt), TransactionAmount.md `WITHHOLDING` formula reads it and
freezes the applied value in the witness.

### InstrumentType becomes a table; quotationFactor lives on it (applied 2026-07-03)
A seeded catalog replacing the enum on Instrument, one row per kind, carrying
`quotationFactor`: BOND 0.01 (percent-quoted), EQUITY / FUND / INDEX / CASH / OBL_BRH /
PROVISION 1 (TERM_DEPOSIT the day TDs are modeled). Also gives
TransactionTypeEligibility its FK target.

Applied: InstrumentType.md created (seeded kinds and factors, provision seeds noted),
Instrument.md now FKs it (`instrumentTypeId`), AssetClass.md gained the PROVISION class
(`isAumEligible = false`), instruments README updated. Cosmetic findings D-8 (broken
portfolio link) and D-9 (char(3) snapshot convention note) were fixed in the same
batch.

### Bond / share / fund type parts (direction confirmed)
These instrument models are not developed yet. When they are written they carry the
parameters the posting functions read: coupon rate, schedule, day-count convention
(Actual/365 or 30/360), withholding rate, preferred-share behavior (fixed vs
discretionary dividend).

## Provisions — generalized concept (recorded 2026-07-03)

A provision is money parked in a holding instrument so it is visibly unavailable. Two
families, same mechanics:

- Fee provisions: a fee (e.g. custody fee in a fund book) is provisioned first, then a
  payment transaction settles it.
- Pending-settlement holds: a fund subscription application moves the amount out of cash
  into the holding instrument until the operation executes; available balance excludes it
  automatically because availability reads the cash position.

Report wanted by the owner: outstanding provisioned amounts, by type of operation.
Backbone implementation, no new columns:

- One transaction type per provision purpose (labels are free to create): the report is
  the positions on provision instruments grouped by (portfolio, transaction type), with
  `reportFamily = PROVISION` selecting them all.
- Stamp the constitution's `eventId` on its release / payment: open provisions are the
  eventId groups whose provision-instrument legs do not sum to zero, and aging falls out
  of `tradeDate`.
- When the position line itself must be distinct (e.g. fund NAV treats a fee provision
  and pending subscriptions differently), use separate holding instruments per bucket.

Instrument pieces applied 2026-07-03: PROVISION kind in InstrumentType.md, PROVISION
asset class (`isAumEligible = false`) in AssetClass.md, PROV-HTG / PROV-USD seeds noted
(price constant 1, issuer ProFin, never lots). Per-purpose holding instruments remain a
case-by-case choice when a position line must be distinct.

## Open decisions

- **CASH_DEPOSIT worked-example leg signs** (TransactionTypeLeg.md): the example shows
  both legs `+` (client book and TRANSACTION_BANK house book both receiving); the
  owner describes the deposit as credit-client / debit-issuer-side. Reconcile the
  intended sign convention for the house-side leg when the owner rules.

## Deferred by decision (2026-07-03)

- ~~Market-data table~~ — **done 2026-07-08**: the market-data/ module
  (InstrumentPrice, FxRate). See the decision entry above.
- The bond / preferred / TERM_DEPOSIT type parts: designed with their modules; they will
  carry the parameters the posting functions read (coupon rate, schedule, day-count,
  preferred-share behavior).
- transaction-module-spec.md will not be written: its intended content (FX sourcing,
  cost-method strategy, backdating, rounding) is stated directly in the table docs; the
  stale reference in the transactions README was removed. The dedicated workflow
  document (POST / APPROVE / REVERSE sequences, full walk trace) remains planned.

## Parked (to review together later)

All findings from the 2026-07-03 verification are now resolved. D-5 closed by decision:
the module spec will not be written; the stale reference was trimmed from the
transactions README (the workflow document remains planned).
Resolved: D-1 and D-7 (RESTATE), D-6 (count fixed in passing), D-3 (cash-instrument
resolution: trade legs read `settlementCashInstrumentId`, pure cash types pick the cash
instrument as the deal's instrument, bank legs use the bank account's currency),
D-2 (required-iff fixed alongside the TRANSACTION_COUNTERPARTY_BANK drop), D-4 (FX
lands as a cross-currency cash trade with a header instrument; every type validates
through eligibility), D-8 and D-9 (fixed with the instrument batch: portfolio link
corrected, char(3) snapshot convention stated on TransactionLeg).
