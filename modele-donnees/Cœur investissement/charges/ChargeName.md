# ChargeName

The **catalog** — the vocabulary of charges: `COMMISSION`, `TCA`, `FEE`,
`COMMISSION_EMETTEUR`, `FRAIS_SID`, … Unlike the nine witness names (closed set, a
tenth is a code change), charge names are **open data**: a new charge is a row,
admin-creatable. Everything the engine needs to move a charge's money lives on the
name itself — which is what makes charge legs fully automatic.

## Essential fields

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `code` | string(50) | ● | Stable machine code — `COMMISSION`, `TCA`, `FRAIS_SID`. Unique. |
| `labelFr` / `labelEn` | string | ● | Labels (entry form field, statements). |
| `precision` / `scale` | int / int | ● | Entry validation and the rounding applied once at computation (legs store `decimal(28,8)` regardless — this is a rule, not a column type). Default: the currency's minor units. |
| `taxChargeNameId` | FK→ChargeName (self) | ○ | **The tax this charge generates** (e.g. `COMMISSION` → `TCA`). The tax follows the *service*, not the tariff: whenever a value under this name is written (manual or policy), the engine derives `value × taxRate` under the tax name, same transaction. `NULL` = untaxed. A name that is itself a tax target cannot carry a binding — **no tax on tax**. |
| `taxRate` | decimal(9,6) | ○ | Required with `taxChargeNameId`. Defined once, here — never per policy. |
| + envelope | | ● | See [README](README.md). |

## Notes & rules

- **No `isTax` flag.** Being the target of a `taxChargeNameId` is the only tax-ness
  that exists: accounting binds by name, tax reporting queries the charge legs by name.
- **No destination book.** A charge is one leg — − the client's settlement cash. The
  receiving side (commission revenue, TCA payable) is the Accounting module's product
  at the GL, bound by charge name; no house book receives a leg.
- A tax-target name is never declared on a type
  ([TransactionTypeCharge](../transactions/TransactionTypeCharge.md)) and never output
  by a policy effect — it exists on transactions only by derivation.
- Retirement is `isActive = false`: no new entries or policies may use the name;
  posted legs keep it forever.

## Clean model

```
ChargeName
  id                      uuid    PK
  code                    string  unique
  labelFr                 string
  labelEn                 string
  precision               int              -- validation + rounding rule, not a column type
  scale                   int
  taxChargeNameId         FK ChargeName?   -- the tax this charge generates; no tax on tax
  taxRate                 decimal(9,6)?    -- required with taxChargeNameId
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)
```
