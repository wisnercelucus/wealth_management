# ChargePolicy

The **tariffs** — the automatic side of charging, no human involved. A policy reads a
base, applies its calculation, and puts the result into a charge name: trigger
(*when*) + [condition](ChargePolicyCondition.md) (*on whom*) +
[effect](ChargePolicyEffect.md) (*how much, into what*).

## Essential fields

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `code` | string(50) | ● | Unique. |
| `labelFr` / `labelEn` | string | ● | |
| `triggerKind` | enum | ● | `TRANSACTION` (a matching transaction posts) \| `SCHEDULE` (the scheduler fires per period). Decides **where the charge lands** — see below. |
| `frequency` | enum | ○ | PeriodFrequency; required **iff** `SCHEDULE`. |
| `conditionId` | FK→ChargePolicyCondition | ● | Named, reusable scope. |
| `effectId` | FK→ChargePolicyEffect | ● | Named, reusable: base → calculation → output charge name. |
| `chargeTransactionTypeId` | FK→TransactionType | ○ | Required **iff** `SCHEDULE`: the type of the charge transaction the policy posts. Typically no contract lines and no declared amounts — its only legs are the auto-generated charge legs. `TRANSACTION` policies need none: their charges land on the trigger. |
| `effectiveFrom` / `effectiveTo` | date | ●/○ | Validity. What applied on a given day is reconstructable from these — which is why no posted row stores a policy id. |
| + envelope | | ● | See [README](README.md). |

## Where the charge lands

- **`TRANSACTION`** — the policy runs **inside the posting block** of every matching
  transaction (operator-entered or emitter-posted alike), after the witnesses are
  computed — its base reads them — and its charge + derived tax legs are generated
  **on the triggering transaction itself**. No second transaction, no `eventId`
  pairing, nothing to reconcile.
- **`SCHEDULE`** — there is no transaction to ride: the policy posts **its own**
  charge transaction per charged book and period, typed by
  `chargeTransactionTypeId`. Header: portfolio = the charged book, instrument = its
  settlement book's declared cash instrument (the book's own when it self-settles —
  `FUND` books declare one); provenance and idempotency via
  `sourceSystem = CHARGE_ENGINE` + `externalId` = the run key — the same emitter
  mechanism as every module.

**A policy fires at most once per event.** Reversing a transaction reverses the
tariff legs it carries; a scheduled charge transaction reverses on its own.

## Manual and automatic never conflict

A policy never pre-fills, suppresses, or reads a manual field
([TransactionTypeCharge](../transactions/TransactionTypeCharge.md)). If the operator deliberately
enters a commission on a transaction a tariff also hits, that is two charges,
additively — the desk knows its tariffs; the system does not police intent. The entry
form may display the tariffs that will fire as a read-only notice (information, never
interaction).

## Notes & rules

- **Authoring guard**: two active policies whose conditions can overlap on the same
  output charge name and trigger are rejected at authoring — double-tariffing is a
  configuration error, caught at save, never discovered at posting.
- The effect's output name must be declared… nowhere: tariff charges need no
  declaration on any type.
- Tax needs no policy: the output name's `taxChargeNameId` binding derives it — a
  `TCA` policy is rejected at authoring (tax-target names are never outputs).
- Retirement is `isActive = false` or a closed `effectiveTo`; posted legs are frozen
  truth either way.

## Clean model

```
ChargePolicy
  id                       uuid    PK
  code                     string  unique
  labelFr                  string
  labelEn                  string
  triggerKind              enum (TRANSACTION | SCHEDULE)
  frequency                enum?             -- required iff SCHEDULE
  conditionId              FK ChargePolicyCondition
  effectId                 FK ChargePolicyEffect
  chargeTransactionTypeId  FK TransactionType?  -- required iff SCHEDULE
  effectiveFrom            date
  effectiveTo              date?
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)
```
