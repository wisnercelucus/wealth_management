# ChargePolicyEffect

A **named, reusable computation** — *how much, into what*. Like conditions, effects
are identified objects: many policies point at the same effect ("1% of PRINCIPAL →
COMMISSION" reused across tariffs whose only difference is the scope).

## Essential fields

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `code` | string(50) | ● | Stable identifier — `PCT1_PRINCIPAL_COMMISSION`. Unique. |
| `name` | string | ● | Display label. |
| `baseKind` | enum | ● | `TRANSACTION_AMOUNT` (a witness of the triggering transaction — `TRANSACTION` policies) \| `POSITION_VALUE` (the charged book's market value as-of period end — `SCHEDULE` policies) \| `FIXED_AMOUNT` (no base). |
| `baseAmountName` | enum | ○ | Required **iff** `TRANSACTION_AMOUNT`: which witness (nine-name enum — `PRINCIPAL`, `GROSS`, `BASE_INTEREST`, …). |
| `method` | enum | ● | `RATE` (MVP: value = base × `rate`) \| `FLAT` (value = `flatAmount`; `CHECK method = FLAT ⟺ baseKind = FIXED_AMOUNT`) \| `BANDED` (reserved). |
| `rate` | decimal(9,6) | ○ | Required iff `RATE`. |
| `flatAmount` / `flatCurrency` | decimal(28,8) / char(3) | ○ | Required iff `FLAT`. |
| `minAmount` / `maxAmount` | decimal(28,8) | ○ | Floor / cap, applied after computation. |
| `outputChargeNameId` | FK→ChargeName | ● | **Where the result goes** — the charge name whose legs the engine generates. Never a tax-target name (tax derives from the output's own binding). |
| + envelope | | ● | See [README](README.md). |

## Notes & rules

- **The base is chosen deliberately, not defaulted.** The BRH maturity commission
  reads `BASE_INTEREST`, never `GROSS`: the index spread is excluded from the
  commission base even though the client is paid `GROSS` when the index triggers.
- The computed value is rounded once, per the output name's `precision`/`scale`, and
  frozen in the legs; `minAmount`/`maxAmount` apply before rounding.
- Editing an effect (a rate change) applies to future postings only — posted legs are
  frozen truth; the applied figure needs no policy pointer to be defensible
  (validity dates on [ChargePolicy](ChargePolicy.md) reconstruct what ruled that day).

## Clean model

```
ChargePolicyEffect
  id                  uuid    PK
  code                string  unique
  name                string
  baseKind            enum (TRANSACTION_AMOUNT | POSITION_VALUE | FIXED_AMOUNT)
  baseAmountName      enum?             -- required iff TRANSACTION_AMOUNT; nine-name enum
  method              enum (RATE | FLAT | BANDED)   -- BANDED reserved; FLAT ⟺ FIXED_AMOUNT
  rate                decimal(9,6)?     -- required iff RATE
  flatAmount          decimal(28,8)?    -- required iff FLAT
  flatCurrency        char(3)?
  minAmount           decimal(28,8)?
  maxAmount           decimal(28,8)?
  outputChargeNameId  FK ChargeName     -- never a tax-target name
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)
```
