# DRAFT — SUPERSEDED 2026-07-08

> **This draft is closed.** The module is finalized in
> [`../charges/`](../charges/README.md) (README, ChargeName, TransactionTypeCharge,
> ChargePolicy, ChargePolicyCondition, ChargePolicyEffect). Late decisions that are
> **not** reflected below: TransactionChargeAmount dropped (charges are legs, stamped
> `chargeNameId`); `destinationPortfolioId` nullable; TRANSACTION-triggered policies
> land on the triggering transaction; manual and automatic strictly separate and
> additive, no pre-fill. See FOLLOW-UP.md (2026-07-08 entry). This file can be deleted.

# DRAFT v3 — Charges module (frais)

Working draft, restructured 2026-07-08 after the owner's direction, revised same day to
resolve charge-leg automation and the BRH commission base. The principle:
**charges are named amounts with their own catalog**, declared per transaction type,
materialized on transactions in their own amount table, fed either by the operator
(manual entry fields) or by charge policies (automatic). A policy-driven charge arrives
as **its own transaction**. The ledger stays the single computable source; nothing
charge-related lives outside it once posted.

## The pieces

```
ChargeName                    -- the catalog: the vocabulary of charges (open, admin-creatable)
TransactionTypeCharge         -- which charge names a transaction type declares (the UI fields)
TransactionChargeAmount       -- the instance rows, linked to a transaction (like TransactionAmount)
ChargePolicy                  -- automatic charges: trigger + condition + effect
  ├── conditionId → ChargePolicyCondition   -- named, reusable scope pattern
  └── effectId    → ChargePolicyEffect      -- named, reusable: base + calc → output charge name
```

Two hard rules the flows obey:

- **A policy always posts its own transaction** — it never writes into the transaction
  that triggered it. Only UI-entered amounts land directly in the triggering
  transaction's TransactionChargeAmount.
- **Posted truth is immutable**: changing TransactionTypeCharge, a policy, a condition
  or an effect never modifies any posted transaction — its charge amounts and legs are
  frozen at posting, like everything else in the ledger.

## ChargeName — the catalog

The set of charge names: `COMMISSION`, `TCA`, `FEE`, `COMMISSION_EMETTEUR`,
`FRAIS_SID`, … Unlike the nine transaction amount names (closed set, a tenth is a code
change), charge names are **open data**: a new charge is a row.

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `code` | string(50) | ● | Stable code — `COMMISSION`, `TCA`, `FRAIS_SID`. Unique. |
| `labelFr` / `labelEn` | string | ● | Labels (entry form field, statements). |
| `precision` / `scale` | int / int | ● | The numeric type of the value, e.g. (7,3) — selected at creation. |
| `taxChargeNameId` | FK→ChargeName (self) | ○ | **The tax this charge generates** (e.g. COMMISSION → TCA). The tax follows the *service*, not the tariff: whenever a charge amount with this binding is written (manual or policy), the engine derives the tax row = `value × taxRate` under the tax name, same transaction. `NULL` = untaxed. A name that is itself a tax target cannot carry a binding (no tax on tax). |
| `taxRate` | decimal(9,6) | ○ | Required with `taxChargeNameId`. Defined once, here — not per policy. |
| `destinationPortfolioId` | FK→Portfolio | ● | **The book that receives this charge** — the house income book for `COMMISSION`/`FEE`, the tax book for `TCA`. Fixed per name, not per type or policy: the engine's auto-generated credit leg always lands here (see [How the money moves](#how-the-money-moves-resolved-automatic-charge-legs)). |
| + envelope | | ● | |

No `isTax` flag: being the target of a `taxChargeNameId` is the only tax-ness that
exists. Routing to the tax book is not a contract-line's job — it is the same
`destinationPortfolioId` mechanism every charge uses; accounting binds by name, and tax
reporting queries the name.

## TransactionTypeCharge — what a type declares

Which charge names a transaction type carries — **this is what makes charge fields
appear in the UI** when the type is selected. On type creation the default set is
seeded (`COMMISSION`, `FEE`); the admin removes or adds names per type. Tax names are
**never declared**: they are derived from their parent charge's `taxChargeNameId`
binding, never entered.

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `transactionTypeId` | FK→TransactionType | ● | The owning type. |
| `chargeNameId` | FK→ChargeName | ● | The declared charge. |
| `sequence` | int | ● | Display order on the form. |
| + envelope | | ● | |

`UNIQUE (transactionTypeId, chargeNameId)`.

**Immutability**: editing these rows never touches posted transactions — their charge
amounts and legs are already materialized. Because charge money movement is automatic
— off the charge name's own `destinationPortfolioId`, never a contract line (see
[How the money moves](#how-the-money-moves-resolved-automatic-charge-legs)) — adding or
removing a declared charge on a type **never** requires a new type code: it only
changes which field appears on the form for future entries. Only a change to the
type's *own* contract lines or `declaredAmounts` still forces a new code.

## TransactionChargeAmount — the instance rows

The charge mirror of TransactionAmount: one row per named charge the transaction
carries, written at posting **after** the witnesses (stage 1), frozen forever. A row
exists only when the charge does — no zero-filling.

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `transactionId` | FK→Transaction | ● | The transaction carrying this charge. |
| `chargeNameId` | FK→ChargeName | ● | Which charge. |
| `value` | decimal | ● | Rounded once at computation (scale from ChargeName). |
| `currency` | char(3) | ● | Snapshot, same convention as legs. |
| `chargePolicyId` | FK→ChargePolicy | ○ | Provenance: set when a policy computed it; `NULL` = operator-entered. |
| + envelope | | ● | Append-only: frozen at posting. |

`UNIQUE (transactionId, chargeNameId)`.

## ChargePolicy — the automatic side

A policy has **an input and an output**: it reads a base amount, applies its
calculation, and puts the result **into a charge name**.

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `code` | string(50) | ● | Unique. |
| `labelFr` / `labelEn` | string | ● | |
| `triggerKind` | enum | ● | `TRANSACTION` (a matching transaction posts) \| `SCHEDULE` (the scheduler fires at `frequency`). |
| `frequency` | enum | ○ | PeriodFrequency; required **iff** `SCHEDULE`. |
| `conditionId` | FK→ChargePolicyCondition | ● | Named, reusable scope (below). |
| `effectId` | FK→ChargePolicyEffect | ● | Named, reusable effect: input → calculation → output charge name (below). |
| `chargeTransactionTypeId` | FK→TransactionType | ● | The type of the **charge transaction** this policy posts (see flow). That type declares the output charge name(s) and its contract lines move the money. |
| `effectiveFrom` / `effectiveTo` | date | ●/○ | Validity. |
| + envelope | | ● | |

## ChargePolicyCondition — named, reusable scope

Unchanged from v1: `code`, `name`, four optional FK axes (`transactionTypeId`,
`partyGroupId`, `portfolioGroupId`, `instrumentGroupId`), set FKs **AND**-ed, `NULL` =
any, `CHECK` at least one set. Many policies point at one condition.

## ChargePolicyEffect — named, reusable: input → calculation → output

Like conditions, effects are identified objects: many policies can point at the same
effect ("1% of PRINCIPAL → COMMISSION" reused across tariffs whose only difference is
the condition).

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `code` | string(50) | ● | Stable identifier, e.g. `PCT1_PRINCIPAL_COMMISSION`. Unique. |
| `name` | string | ● | Display label. |
| `baseKind` | enum | ● | `TRANSACTION_AMOUNT` \| `POSITION_VALUE` \| `FIXED_AMOUNT`. |
| `baseAmountName` | enum | ○ | Required iff `TRANSACTION_AMOUNT`: the witness (`PRINCIPAL`, `GROSS`, …). |
| `method` | enum | ● | `RATE` (MVP). `FLAT_AMOUNT`, `BANDED` reserved. |
| `rate` | decimal(9,6) | ○ | Required iff `RATE`. |
| `flatAmount` / `flatCurrency` | decimal / char(3) | ○ | For `FIXED_AMOUNT`. |
| `minAmount` / `maxAmount` | decimal(28,8) | ○ | Floor / cap. |
| `outputChargeNameId` | FK→ChargeName | ● | **The place the result goes** — must be declared by the policy's `chargeTransactionTypeId`. |
| + envelope | | ● | |

## The two flows

**Manual (declared on the type).** The operator selects `CASH_WITHDRAWAL`; the type
declares `FEE` → one charge field appears on the form (the TCA shows as a derived,
read-only preview beside it). He enters (or leaves empty) the value. At posting:
witnesses computed first (stage 1), then the entered charge is written as a
TransactionChargeAmount row on the **same transaction**, its TCA row derived from the
name's tax binding, then legs. Empty = no row = no charge, no tax.

**Automatic (policies).** A policy fires on its trigger:

- `TRANSACTION`: a matching transaction posts (e.g. `BRH_MATURITY_INTEREST`) → the
  policy computes (base from the trigger transaction's witnesses) and posts **its own
  charge transaction** — typed by `chargeTransactionTypeId`, carrying its
  TransactionChargeAmount rows (`COMMISSION`, chained `TCA`) and the legs that move the
  money — grouped with the trigger under one `eventId`.
- `SCHEDULE`: the scheduler fires per period → same thing, base = `POSITION_VALUE` or
  `FIXED_AMOUNT`, one charge transaction per charged book and period.

**Tax derivation.** No chaining needed: whenever a charge amount is written and its
name carries a `taxChargeNameId`, the engine writes the tax row (`value × taxRate`) in
the same transaction. Waiving / omitting the parent means no tax row exists either.

## How the money moves (resolved: automatic charge legs)

A charge amount row is a **figure**; money moves through legs, same as everywhere else
— but a charge never needs an authored contract line. The type author declares *which*
charges exist (`TransactionTypeCharge`); the engine decides *how* they move, always the
same way, off the charge name itself:

**Every `TransactionChargeAmount` row generates exactly two legs, automatically — no
contract line involved:**

```
−  TRANSACTION_CASH                    ·  CASH  ·  chargeName   (the client's settlement side pays)
+  chargeName.destinationPortfolioId   ·  CASH  ·  chargeName   (the charge's own book receives)
```

This reverses the earlier proposal to extend `TransactionTypeLeg.amountName` to the
charge vocabulary. That path meant an author hand-writing two (or four, with tax)
conditional lines per declared charge on every type, kept in lockstep with
`TransactionTypeCharge` — exactly the sprawl and mismatch risk flagged: a user
declaring a charge on a type would *also* have to carefully author and align matching
contract lines, on every type, forever. Instead the destination is a property of the
**charge name itself** (`destinationPortfolioId`), so:

- Declaring a charge on a type is now **purely additive** — one `TransactionTypeCharge`
  row, no matching contract lines to author, nothing to keep in sync.
- Because charge money movement never touches the type's own (frozen) contract lines,
  **adding a declared charge to a type that has already posted transactions is safe** —
  no new type code required (see the immutability note above).
- The chained tax leg (`TCA`, when a parent's `taxChargeNameId` fires) follows the exact
  same two-leg rule, off `TCA`'s own `destinationPortfolioId` (the tax book).

Posting order:

```
1  witnesses            → TransactionAmount rows (unchanged)
1b charge amounts       → TransactionChargeAmount rows: UI-entered or policy-computed,
                          then derived tax rows (same chaining rule)
1c charge legs          → automatic, per the two-leg rule above — no contract lines,
                          no authoring, one row in ⇒ two legs out, every time
2  legs                 → TransactionTypeLeg contract lines, witnesses only (unchanged)
3  lot effects, etc.    → unchanged
```

- a manual withdrawal fee = one declared charge (`FEE`) on `CASH_WITHDRAWAL`, **zero**
  authored lines; posting writes the `FEE` amount + its derived `TCA` amount, then
  their four legs, automatically;
- a policy's charge transaction = the same mechanism, just triggered by the policy
  instead of operator entry — `chargeTransactionTypeId` declares the output charge
  name(s) via `TransactionTypeCharge` and needs **no contract lines of its own** either.

This keeps "positions = Σ legs" absolute and removes both the separate settlement
machinery *and* the contract-line authoring burden for charges.

## Worked examples

```
ChargeName seeds: TCA · COMMISSION (tax → TCA, 10%) · FEE (tax → TCA, 10%)
                  · COMMISSION_EMETTEUR · FRAIS_SID

CASH_WITHDRAWAL (type)               -- declares FEE → one field in the UI; TCA derived
  own economics (authored contract lines):
        1 CARRY · TRANSACTION      · TRANSACTION · − · PRINCIPAL
        2 CARRY · TRANSACTION_BANK · TRANSACTION · − · PRINCIPAL
  charge legs (automatic — no authoring, fires only if the field was filled):
        − TRANSACTION_CASH               · CASH · FEE   /  + FEE.destinationPortfolioId · CASH · FEE
        − TRANSACTION_CASH               · CASH · TCA   /  + TCA.destinationPortfolioId · CASH · TCA

COMM_BRH_MATURITY_RETAIL1 (policy)   -- automatic commission at BRH maturity
  trigger    TRANSACTION
  condition  RETAIL1_BRH_MATURITY    (type = BRH_MATURITY_INTEREST AND partyGroup = Retail 1)
  effect     base TRANSACTION_AMOUNT·BASE_INTEREST → RATE x% → output COMMISSION
             (never GROSS — the index spread, when the index triggers, is excluded
             from the commission base)
  posts      charge transaction (type CHARGE_BRH_COMMISSION, no contract lines needed —
             its legs are automatic), eventId = the maturity's
             TCA row derived automatically from COMMISSION's tax binding — no TCA policy
```

## Open decisions

1. ~~Charge-fed legs~~ — **resolved, reversed**: no extension of
   `TransactionTypeLeg.amountName`. Charge legs are **automatic**: two legs per
   `TransactionChargeAmount` row, keyed off the charge name's own
   `destinationPortfolioId` (client cash out, the charge's own book in). No authoring,
   no lockstep with `TransactionTypeCharge`, no per-type contract-line sprawl the
   admin would have to configure carefully alongside every declared charge.
2. ~~Waive~~ — **resolved**: no waive status exists. Absence is the waiver: an empty
   field at entry = no row = no charge, no tax. A permission-gated "clear" on
   pre-filled fields is an optional UI nicety. After posting, reversal only.
3. **Pre-fill** — may a policy pre-fill a *declared* manual field (operator confirms,
   deviation stamped, fxRate-style), or are manual and automatic strictly separate?
4. ~~Type freeze~~ — **resolved**: posted transactions are never modified by relation
   changes (truth materialized). Editing `TransactionTypeCharge` now affects future
   entries only, full stop — since charge legs are automatic (not contract lines),
   declaring or removing a charge on an already-posted type **no longer requires a new
   type code** either.
5. ~~BRH commission base~~ — **resolved**: the fixed part only, `BASE_INTEREST`. The
   index spread (`INDEX_SPREAD`) is never part of the commission base, even though the
   client is paid `GROSS` (`BASE_INTEREST + INDEX_SPREAD`) once the index triggers.
6. **Impacts to apply once agreed** — TransactionType.md (declared charges),
   ChargeName's new `destinationPortfolioId` field, TransactionAmount.md
   admission-test wording (charges live in TransactionChargeAmount, still never
   witnesses), transactions README boundary. `TransactionTypeLeg.md` needs **no**
   change — charges never touch it.
