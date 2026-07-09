# AssetClass

A classification that groups instruments by how they behave economically. It is used for two
things: how holdings are grouped in reports (the allocation view), and whether a holding counts
toward AUM.

Asset class is deliberately **coarse** and **controlled**. The classes are a stable, near-universal
vocabulary, extended only by an administrator on the rare occasion ProFin genuinely takes on a new
kind of class — it is not a list business users add to freely. Keeping it small and stable is what
stops it drifting into a second, redundant way of typing instruments: an instrument's *type* says
*what it is*; its *asset class* says *how it behaves*. The two are separate, which is why an
instrument's asset class can group it with instruments of a different type that behave the same way.

> Assumes the shared envelope (see [README](./README.md)).

## Fields

| Field | Type | Req | Meaning |
|---|---|---|---|
| `id` | uuid/bigint | ● | Surrogate PK. |
| `code` | string | ● | The asset class identifier. Join on `id`, not this. |
| `name` | string | ● | Display label. |
| `isAumEligible` | bool | ● | Whether holdings of this class count toward AUM. |
| + envelope | | ● | see README. |

## Values

| `code` | `isAumEligible` | Meaning |
|---|---|---|
| `EQUITY` | true | Ownership positions; return comes from growth and dividends. |
| `FIXED_INCOME` | true | Positions whose return is a contractual rate of interest. |
| `FUNDS` | true | Holdings in pooled funds. |
| `CASH` | false | Liquidity held in a portfolio — reported on its own line, not as invested AUM. |
| `PROVISION` | false | Money parked in a holding instrument (fee provisions, pending-settlement holds) — visibly unavailable, reported on its own line, never AUM. |
| `BENCHMARK` | false | A reference series used for valuation and protection, not held as an investment. |

## AUM eligibility

Whether a class counts toward AUM is read from `isAumEligible` on the class, not from a fixed list
of class names. So a class added later carries its own eligibility and is taken into account
automatically. A holding counts toward AUM when its instrument's class is AUM-eligible — together
with the portfolio-level conditions defined in the Portfolio model (which decide *which books*
count, before the asset class decides *which holdings* within them).
