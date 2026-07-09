# TransactionType

The **catalogue row** — a pure declaration. What the operator picks from a menu. A type
declares its structure ([contract lines](TransactionTypeLeg.md)) and its amounts
(`declaredAmounts`); *how* an amount is computed never lives here — header-derived
amounts are generic arithmetic, instrument-derived amounts belong to the instrument
type's [calculator](../instruments/InstrumentCalculator.md). A new *label* is a row
(free, admin-creatable); a new *instrument behaviour* is a registered calculator
(deliberate, developer-sized). That friction gradient is what prevents the 243-type
sprawl of the legacy system.

## Essential fields

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `code` | string(50) | ● | Stable machine code — `BUY_BOND`, `CASH_DEPOSIT`. Unique; application logic keys off it. |
| `labelFr` | string | ● | French label (UI, statements). |
| `labelEn` | string | ● | English label. |
| `isSystem` | bool | ● | `true` → seeded, undeletable, contract lines read-only. |
| `requiresChecker` | bool | ● | Feeds the Approval module's policy (which can add its own thresholds — e.g. checker required for any backdated entry). |
| `declaredAmounts` | jsonb | ● | The amount-output contract of **every** type — array of `{name, required}` over the nine names, e.g. `BOND_COUPON: [GROSS ✓, WITHHOLDING ○, INCOME ✓]`. The gate validates **both directions**: required-but-absent → reject; produced-but-undeclared → reject. Eligibility is selected on the same authoring screen, so every save validates each instrument-derived name against what the eligible instrument types can compute (see [InstrumentCalculator](../instruments/InstrumentCalculator.md)). |
| `reportFamily` | enum | ● | Reporting classification: `CASH \| SECURITY_TRADE \| INCOME \| TRANSFER \| FX \| MATURITY \| CORPORATE_ACTION \| PROVISION \| CHARGE_SETTLEMENT`. |
| `cashFlowCategory` | enum | ● | `EXTERNAL_IN \| EXTERNAL_OUT \| INTERNAL` — **what MWR/TWR cannot live without**: external flows vs internal movements. A column, not a hardcoded type-code list, so admin-created types classify themselves. |
| `isClientVisible` | bool | ● | Whether transactions of this type appear on client statements and the portal (provisions and omnibus moves don't). |
| `narrativePattern` | string | ○ | Statement text template — `"Achat {qty} {instrument} @ {price}"`. |
| `description` | text | ○ | Operational guidance. |
| + envelope | | ● | See [README](README.md). `externalId` = the legacy Axia type code this replaces — the migration map. |

## Types are declaration

A type carries **no behavior**. Its [contract lines](TransactionTypeLeg.md) define the
structure (which legs, from which addresses); `declaredAmounts` defines which amounts
exist. *How* each amount is computed lives elsewhere, routed by the amount name itself:

- **Header-derived names** (`QUANTITY`, `PRINCIPAL = qty × price × quotationFactor`,
  `ACCRUED = input`, `COUNTER = settling cash × fxRate`) — computed by the generic
  engine from the header's five inputs. Always available to every type.
- **Instrument-derived names** (`GROSS`, `WITHHOLDING`, `INCOME`, `BASE_INTEREST`,
  `INDEX_SPREAD`) — computed by the eligible instrument type's registered
  [calculator](../instruments/InstrumentCalculator.md), which reads instrument
  parameters (coupon rate, day-count, reference index, `withholdingRate`).

An admin authors any type end to end: the authoring UI offers the four header-derived
names plus whatever the eligible instrument types can compute. Declaring `INDEX_SPREAD`
on a type eligible for plain bonds is rejected at type-creation ("ordinary bonds do not
compute INDEX_SPREAD"), never at posting. Stages 2–4 of posting (legs → lot effects →
position) are one generic engine for every type, forever — no code ever builds legs.

## Type immutability (no versioning)

Once a type has posted transactions, its contract lines and `declaredAmounts` are
**frozen**. Changed economics = a **new type code**; changed calculator math = a new
`functionKey` in the [calculator registry](../instruments/InstrumentCalculator.md)
(code history lives in git). No version tables: a
transaction's truth is fully materialized in its witnesses and legs — nothing is ever
re-derived from the contract. Presentation attributes (labels, `narrativePattern`,
`requiresChecker`, the reporting columns) stay freely editable.

## Notes & rules

- **No `economicEffect`.** An earlier draft carried one; it was cut for having no
  consumer, and revived only as the three reporting columns above — each of which has a
  named consumer (MWR, statements, the portal). A column earns its place when something
  computes from it.
- **No `journalTemplateId`.** The Accounting module binds *its* tables to type codes;
  the dependency points accounting → ledger, never the reverse.
- **Declared charges** are the type's manual charge fields
  ([TransactionTypeCharge](TransactionTypeCharge.md)): which charge names
  the entry form offers the operator. They are not contract lines — charge legs are
  auto-generated by the engine — so declaring or removing one on a type that has
  already posted changes future entries only, no new type code. Tariff charges
  ([ChargePolicy](../charges/ChargePolicy.md)) need no declaration at all.
- Retirement is the envelope's `isActive = false` — history keeps its type.

## Clean model

```
TransactionType
  id                  uuid    PK
  code                string  unique
  labelFr             string
  labelEn             string
  isSystem            bool             -- seeded, undeletable, contract lines read-only
  requiresChecker     bool
  declaredAmounts     jsonb            -- every type: [{name, required}] over the nine names
  reportFamily        enum (CASH | SECURITY_TRADE | INCOME | TRANSFER | FX |
                            MATURITY | CORPORATE_ACTION | PROVISION | CHARGE_SETTLEMENT)
  cashFlowCategory    enum (EXTERNAL_IN | EXTERNAL_OUT | INTERNAL)
  isClientVisible     bool
  narrativePattern    string?
  description         text?
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)
  -- frozen once posted transactions exist: contract lines, declaredAmounts
```
