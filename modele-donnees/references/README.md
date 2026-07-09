# Référentiel — Reference-data / shared-kernel layer

The shared primitives the whole platform depends on. **Every domain imports this layer; this
layer imports none of them** — it sits at the bottom of the dependency graph, which is what
makes it safe to depend on everywhere.

**Membership rule** — an entity lives here only if all three hold: (1) two or more domains
reference it, (2) no single domain owns it, (3) it's slowly-changing reference data, not
transactional.

## Tables (5)

| Table | Purpose | Behavior |
|-------|---------|----------|
| [Currency](Currency.md) | Currencies ProFin operates in (HTG/USD/EUR) | trivial |
| [Country](Country.md) | ISO country list — addresses, domicile, nationality | trivial |
| [Calendar](Calendar.md) | Named business-day calendars (Haiti, US) | business-day logic |
| [CalendarHoliday](CalendarHoliday.md) | Holiday dates per calendar | — |
| [CurrencyPair](CurrencyPair.md) | FX pair definition (base/quote) — **not** the rate | FX direction |

## Enums (4) — code-level, see [enums.md](enums.md)

`Language` (FR\|EN) · `BusinessDayConvention` · `DayCountConvention` · `PeriodFrequency`.

Each is kept here because it's referenced across several domains. Enums used by a **single**
table are **not** here — `AddressType` (only `PartyAddress`) and `ContactMethod` (only
`PartyContact`) are defined locally in the Party cluster.

## Not here — usage links live in their domain

`PartyAddress` (party ↔ address) and `PartyContact` (party emails/phones) are **not** in this
layer — they FK `Party`, and a reference primitive must FK nothing in the domains, or
`referentiel → Party` and `Party → referentiel` form a cycle. They live in the **Party
cluster** with their fields inline (no shared `Address` primitive). The pattern throughout: *the
kernel holds shared reference data, the domain holds party-owned data.*

## Internal relationships (no FK leaves this layer)

```
Country  1───* Calendar
Calendar 1───* CalendarHoliday
Currency 1───* CurrencyPair  (base)
Currency 1───* CurrencyPair  (quote)
```

## Conventions

camelCase fields; FK = `<table>Id`; surrogate `id` PK on every table; **join on `id`, never on
`code`**. `Req`: ● essential · ○ optional.

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

> On bulk reference rows (e.g. `CalendarHoliday`) only `isActive` carries real weight; the rest
> of the envelope is uniform by convention.
