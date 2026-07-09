# OBL BRH

BRH bonds — fixed-rate instruments issued by the Banque de la République d'Haïti, with a HTG/USD
protection mechanism. They are modelled by **two entities**: a reusable **template**
(`OblBrhTemplate`) that defines a product type, and the concrete **emission** (`OblBrh`) created
from it. An emission is an instrument; it is one-to-one with the base instrument like any other type.

Although called an "obligation," a BRH bond is its own kind of instrument: it pays in full at
maturity with no periodic coupons, has no secondary market (only early redemption with a penalty),
accrues off a moving benchmark rather than a fixed coupon, uses its own rate-divisor calculation,
and carries a template layer.

## The three tiers: Template → Emission → Subscription

Understanding OBL BRH means seeing three distinct things:

1. **Template** — a product type, e.g. `OBL-BRH-182`. It defines the parameters common to that type
   of BRH bond. It is a reusable definition, not a tradeable instrument.
2. **Emission** — a concrete issue created from a template. It takes the template's parameters as
   its own and adds the specifics of that issue (its dates, its custodian). The emission is the
   instrument clients actually hold.
3. **Subscription** — a client investing is recorded as a transaction against an emission, not as a
   new instrument. This is why one emission can carry many clients' subscriptions, rather than a
   new instrument being created per investment.

The different "BRH models" are simply different **templates** (`OBL-BRH-91`, `-182`, `-364`, …).
The variation between them is data, not different structures.

## OblBrhTemplate

A reusable template for one type of BRH bond. It defines the defaults that each emission takes on
when it is created. It is a **definition, not an instrument**: never held, never priced, never a
position. Importantly, **changing or retiring a template does not affect emissions already
created** — an emission takes a *copy* of the template's values at the moment it is created, so the
template can evolve without rewriting the history of past issues.

> Assumes the shared envelope (see [README](../README.md)). The envelope's `isActive` governs
> whether the template is active — i.e. usable to create new emissions.

| Field | Type | Req | Meaning |
|---|---|---|---|
| `id` | uuid/bigint | ● | Surrogate PK. |
| `code` | string | ● | Template identifier, e.g. `OBL-BRH-182`. Join on `id`, not this. |
| `name` | string | ● | Label, e.g. "OBL BRH 182 jours". |
| `annualRate` | decimal | ● | Default fixed annual rate (the "Custom BRH" rate). |
| `durationDays` | integer | ● | Default term in days (91, 182, 364, …). |
| `rateDivisor` | integer | ● | The divisor used in the daily interest calculation. Configurable, sometimes set by hand. |
| `successPremiumRate` | decimal | ● | The prime de succès rate — ProFin's earning from BRH at maturity, and the amount BRH reclaims (prorated) on early redemption. A defining economic parameter of the product, not a charge. |
| `referenceIndexId` | FK → Instrument (Index) | ● | The benchmark this product tracks — the HTG/USD benchmark instrument. |
| + envelope | | ● | see README. |

## OblBrh (emission)

A concrete BRH bond emission, and the instrument clients hold. It is one-to-one with the base
[Instrument](../Instrument.md). It is created from a template, taking a copy of the template's
parameters, and adds the facts particular to that issue.

> Assumes the shared envelope (see [README](../README.md)).

| Field | Type | Req | Meaning |
|---|---|---|---|
| `id` | uuid/bigint | ● | Surrogate PK. |
| `instrumentId` | FK → Instrument | ● | The base instrument (one-to-one). |
| `templateId` | FK → OblBrhTemplate | ● | The template this emission was created from — kept for provenance. The values below are *copied*, so retiring the template leaves the emission intact. |
| `annualRate` | decimal | ● | Copied from the template, fixed at creation. |
| `durationDays` | integer | ● | Copied from the template. |
| `rateDivisor` | integer | ● | Copied from the template. |
| `successPremiumRate` | decimal | ● | Copied from the template. |
| `referenceIndexId` | FK → Instrument (Index) | ● | Copied from the template — the benchmark this emission tracks. |
| `issueDate` | date | ● | The emission's reference date. The protection baseline is the benchmark's value on this date, read live from the series — not stored. |
| `maturityDate` | date | ● | The issue date plus the term. |
| `custodianId` | FK → Party | ○ | The custodian holding the emission (typically BRH). |
| `minRedemptionDelayDays` | integer | ○ | The minimum holding period before early redemption is allowed. |
| + envelope | | ● | see README. |

## Emission before subscription

An emission exists **before** anyone subscribes to it. BRH announces an issue — its rate, duration,
and dates — and that is the moment the emission is recorded: the operator chooses the template and
enters the dates. Because the emission already exists by the time a client invests, subscribing is
simply a matter of choosing an existing emission; the template is never something an operator picks
when booking a subscription. The same emission can also arrive through migration, carrying its
legacy reference (in the envelope's `externalId`) — once created, a migrated emission and one
created in ProFin are the same kind of record.

## The protection mechanism (the spread)

A BRH bond pays a fixed rate, protected against depreciation of the gourde by the benchmark. The
baseline is the benchmark's value on the emission's issue date, read live from the series:

- If the benchmark has not moved unfavourably (its current value is at or below its value on the
  issue date) → the client receives the **fixed rate**.
- If it has moved unfavourably (its current value is above its value on the issue date) → the
  client receives the **fixed rate plus the benchmark differential**.

The spread is applied at **maturity**. On early redemption — and on any buy/sell — only the **fixed
rate** applies; the spread is set aside. For valuation in between, the rate is estimated daily from
the current benchmark value against the baseline.

## What is computed, not stored

- The **protection baseline** — the benchmark's value on the issue date — is read from the series
  when needed, not stored on the emission, so a later correction to the benchmark flows through to
  every valuation and the maturity calculation automatically.
- The **daily accrual** (estimated from the benchmark) — for valuation.
- The **current position** and the **outstanding subscribed nominal** — from the subscriptions.
- The **early-redemption penalty** — derived from `successPremiumRate`, prorated for the time
  elapsed. There is no separate penalty figure to keep: the penalty *is* the prorated prime de
  succès, a fixed rate that does not vary.

## What lives elsewhere

- **Fees, charges and taxes** (commission, TCA, withholding, any transaction or product fee) are
  configured in the charge matrix and applied at booking — they are not fields on the instrument.
  `successPremiumRate` is the only rate on the instrument, and it is there as a product parameter,
  not as a charge.
- **The success-premium money movement** belongs to the BRH↔ProFin settlement, not to the
  instrument. The instrument carries the *rate*; the actual movement of money (BRH crediting ProFin,
  the debit note, the maturity credit, repatriation) is settlement.
- **The subscription amount and the "new resource" indicator** belong to the subscription
  transaction, not to the instrument.

## Classification

For reporting and AUM, an OBL BRH is classified as [`FIXED_INCOME`](../AssetClass.md) — it behaves as
fixed income.
