# Instrument Cluster

## Purpose

The instrument cluster is ProFin's catalogue of the financial instruments it trades, holds, and
reports on. The objective is a model shaped to ProFin's actual instruments and the way ProFin
operates them, rather than a generic platform that forces ProFin's products into abstractions that
don't fit.

## Approach

Every instrument shares a common identity — what it is, who issued it, its currency, its status —
and each kind of instrument adds the specifics only it needs. So the model is a **base instrument**
plus a **type-specific part** for each kind, one-to-one with the base. A BRH bond, for instance,
carries its own template and protection terms, while the shared identity stays free of anything
particular to one type.

A second principle runs through the cluster: **what can be computed is not stored.** Positions,
balances, and accruals are worked out from the transaction history and the price series — they are
not kept as figures on the instrument. The instrument tables hold *definitions*; the numbers are
produced when reports are read.

## Documents

- **[Instrument](./Instrument.md)** — the shared identity every instrument has.
- **[InstrumentType](./InstrumentType.md)** — the seeded catalog of instrument kinds (quotation factor, kind-level law).
- **[InstrumentCalculator](./InstrumentCalculator.md)** — the registry mapping (instrument type, amount name) to the registered code function that computes it.
- **[AssetClass](./AssetClass.md)** — the classification used for reporting and AUM.
- **[InstrumentGroup](./InstrumentGroup.md)** — free commercial buckets (group + membership), the instrument-side scope of charge and taxation rules.
- **[OBL BRH](./OBL%20BRH/OblBrh.md)** — BRH bonds: the product template and the emission created from it.

## Conventions

camelCase fields; FK = `<table>Id`; surrogate `id` PK; **join on `id`, never on `code`**.
`Req`: ● essential · ○ optional.

## Shared envelope — on every table

```
id          uuid / bigint   PK
isActive    bool            soft-delete — never hard-delete
externalId  string          external-system id (integration / reconciliation)
externalRef string?         external-system reference
createdAt   datetime
createdBy   FK→User
modifiedAt  datetime
modifiedBy  FK→User
```

## Reference FKs (→ référentiel)

`Currency` lives in the reference layer; instruments hold `currencyId`.

## Out of scope (lives elsewhere)

Prices / NAVs / value series → central pricing module; transaction charges and fees → taxation / fees layer
(the instrument keeps only its own `withholdingRate`); maker-checker → approbation module; full audit trail (10-year BRH) → audit
layer (the envelope keeps only created / modified provenance); positions, accruals and valuations →
derived (reporting); BRH↔ProFin money movements → settlement.
