# Instrument (base)

The shared identity that every instrument has, whatever its type — the facts that are **universal**
across all instruments: what it is, who issued it, its currency, and its classification. Anything
specific to a particular kind of instrument lives in that kind's own part, one-to-one with this
base.

The objective is to keep the base to only what is genuinely common, so that the specifics of any
one type never spill onto the shared identity. `instrumentTypeId` points at the
[InstrumentType](./InstrumentType.md) catalog, which records which kind an instrument is, and
therefore which type-specific part carries its detail.

> Assumes the shared envelope (see [README](./README.md)).

## Fields

| Field | Type | Req | Meaning |
|---|---|---|---|
| `id` | uuid/bigint | ● | Surrogate PK. |
| `code` | string | ● | Internal instrument code — present on every instrument (local instruments have no ISIN). Join on `id`, not this. |
| `name` | string | ● | Short name. |
| `description` | text | ○ | Longer description. |
| `instrumentTypeId` | FK → [InstrumentType](./InstrumentType.md) | ● | The kind of instrument (seeded catalog: BOND, EQUITY, FUND, INDEX, CASH, OBL_BRH, PROVISION). Points to the type-specific part that holds the detail, and carries the kind's `quotationFactor`. |
| `assetClassId` | FK → [AssetClass](./AssetClass.md) | ● | The instrument's classification, used for reporting and AUM. |
| `currencyId` | FK → Currency (référentiel) | ● | `HTG \| USD`. |
| `issuerId` | FK → Party | ● | The issuing entity, held as a role on Party. Always present — BRH for BRH bonds, ProFin for cash and benchmarks. |
| `issuerPortfolioId` | FK → Portfolio | ○ | The issuer-side book transactions of this instrument settle into (a `bookKind = ISSUER` house book). An issuer can run several issuer books, one per product family (CIC émetteur / CIC émetteur obligations), so the instrument names its own. Read by the ledger's `TRANSACTION_INSTRUMENT_ISSUER` portfolio mode; required at posting when the type's contract uses that mode (`NULL` → reject at entry). |
| `withholdingRate` | decimal(5,4) | ○ | Fixed at-source withholding rate on this instrument's income (0.15 shares, 0.20 ordinary bonds). **`NULL` = exempt** (OBL BRH). Read at posting by the income functions; the applied value is frozen in the ledger's `WITHHOLDING` witness. |
| + envelope | | ● | see README. |

## Notes

- **Whether an instrument is active** is the envelope's `isActive` — there is no separate status.
- **International identifiers** (ISIN, CUSIP) belong to the instrument types that use them, not to
  the shared base — the base relies on `code`.
- **Prices, NAVs, and value series** are not kept here; they live in the central pricing module.
- **Withholding is a rate on the instrument**, not a client × instrument matrix: ProFin's rates
  are fixed per kind (15% shares, 20% ordinary bonds, BRH exempt). The ledger freezes the applied
  value in each transaction's witnesses, so history survives any later correction of the rate.
- **Audit and approval** — the envelope keeps created / modified provenance; the full audit trail
  and maker-checker control are provided by the core (audit layer, approbation module).
