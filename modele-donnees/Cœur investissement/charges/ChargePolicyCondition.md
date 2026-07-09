# ChargePolicyCondition

A **named, reusable scope** — *on whom* a policy applies. Conditions are identified
objects so many policies can point at one ("Retail 1 on BRH maturities" reused by
every tariff that targets that population); editing the condition retargets them all,
for future postings only.

## Essential fields

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `code` | string(50) | ● | Stable identifier — `RETAIL1_BRH_MATURITY`. Unique. |
| `name` | string | ● | Display label. |
| `transactionTypeId` | FK→TransactionType | ○ | Match the triggering transaction's type. Meaningless on `SCHEDULE` policies (no trigger transaction) — left `NULL` there. |
| `partyGroupId` | FK→PartyGroup | ○ | Match the charged book's owner group. |
| `portfolioGroupId` | FK→PortfolioGroup | ○ | Match the charged book's group. |
| `instrumentGroupId` | FK→InstrumentGroup | ○ | Match the deal instrument's group. |
| + envelope | | ● | See [README](README.md). |

## Notes & rules

- **Set axes are AND-ed; `NULL` = any.** `CHECK`: at least one axis set.
- Group membership is explicit ([PartyGroup](../clients/PartyGroup.md),
  [PortfolioGroup](../portefeuilles/PortfolioGroup.md),
  [InstrumentGroup](../instruments/InstrumentGroup.md)); a rule that must catch every
  instrument of a kind scopes on the kind through its group, maintained deliberately.
- Evaluated at posting time against the transaction (or at run time against the book,
  for `SCHEDULE`); the result is frozen in the legs it produced — editing a condition
  never touches posted truth.

## Clean model

```
ChargePolicyCondition
  id                 uuid    PK
  code               string  unique
  name               string
  transactionTypeId  FK TransactionType?   -- NULL on SCHEDULE-policy conditions
  partyGroupId       FK PartyGroup?
  portfolioGroupId   FK PortfolioGroup?
  instrumentGroupId  FK InstrumentGroup?
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)
  -- set axes AND-ed; NULL = any; CHECK at least one axis set
```
