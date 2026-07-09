# TransactionTypeEligibility

The **permission map** between transaction types and instrument kinds — which types
may be posted on which kinds of instrument. Drives the operator's menu (pick an
instrument, see only eligible types) and the entry gate (an ineligible pairing rejects
before anything else runs).

## Essential fields

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `transactionTypeId` | FK→TransactionType | ● | The type being permitted. |
| `instrumentTypeId` | FK→InstrumentType | ● | The instrument **kind** it is permitted for — kind granularity, never per-instrument (same spirit as charge policies). |
| `isDefault` | bool | ● | Whether this type is the suggested default for the kind (pre-selected in the UI). |
| + envelope | | ● | See [README](README.md). |

## Notes & rules

- One active row per `(transactionTypeId, instrumentTypeId)`.
- **Authored with the type, never after it.** Eligibility is selected on the type's own
  authoring screen, together with `declaredAmounts`: a type cannot be saved without its
  eligible kinds, and every save (creation or later change) re-runs the coherence check.
  Adding a kind that cannot compute one of the type's required amounts is rejected on
  the spot, never at posting (see
  [InstrumentCalculator](../instruments/InstrumentCalculator.md)).
- **This table is where instrument-lifecycle law lives**: OBL BRH has no secondary
  market — no `SELL` or `TRANSFER` row exists for its kind, so the operation is
  impossible at entry, not caught downstream.
- **INDEX is never eligible**: benchmarks are never transacted; creating an eligibility
  row for that kind is rejected.
- Eligibility validates against the header's `instrumentId` (its kind). Pure cash, FX,
  and provision types pick their cash / provision instrument as the deal's instrument
  (`instrumentMode = TRANSACTION`), so **every type** validates the same way — map them
  to the cash / provision instrument kinds.
- Removing an eligibility (envelope `isActive = false`) stops new postings; history
  keeps its rows untouched.

## Clean model

```
TransactionTypeEligibility
  id                 uuid   PK
  transactionTypeId  FK TransactionType
  instrumentTypeId   FK InstrumentType
  isDefault          bool
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)
  unique (transactionTypeId, instrumentTypeId) where isActive
```
