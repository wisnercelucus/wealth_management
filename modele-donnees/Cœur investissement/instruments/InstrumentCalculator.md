# InstrumentCalculator

The **registry mapping an instrument type + an amount name to the code function that
computes that amount**. This is the seam between the data half (which amounts a
transaction type declares) and the code half (how a value is produced). Three things
read it: the amount-generation engine at posting (stage 1 of the pipeline), the
type-authoring UI (to offer only computable amounts), and the authoring-time coherence
check.

**Nature of the table.** Rows are **not freely creatable**: each row names a code
function that must exist. A new row is added only after a developer has written and
registered its function. This is the deliberate friction — a genuinely new instrument
behaviour (a new way to compute an amount) is a code change, not a config action; a new
*instrument* that reuses existing math is just data (a new [Instrument](./Instrument.md)
row on an existing [InstrumentType](./InstrumentType.md)). The registry grows only at
the true frontier: an amount nobody knew how to compute before.

> Assumes the shared envelope (see [README](./README.md)).

## Fields

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `instrumentTypeId` | FK→[InstrumentType](./InstrumentType.md) | ● | The instrument type whose parameters this calculator reads. |
| `amountName` | enum | ● | Which of the instrument-derived amount names this computes (`GROSS`, `WITHHOLDING`, `INCOME`, `BASE_INTEREST`, `INDEX_SPREAD`). Same nine-name vocabulary as the ledger; header-derived names (`QUANTITY`, `PRINCIPAL`, `ACCRUED`, `COUNTER`) never appear here. |
| `functionKey` | string(100) | ● | Stable key of the registered code function (e.g. `bond.computeCoupon`, `oblBrh.computeIndexSpread`). Resolved to real code by the calculator registry; **never a formula string** — the UI selects from registered keys, it does not author formulas. |
| `description` | string | ○ | What the calculator does and which instrument parameters it reads. |
| + envelope | | ● | See [README](./README.md). |

`UNIQUE (instrumentTypeId, amountName)` — one calculator per (type, amount). Sparse by
nature: an instrument type carries rows **only for the amounts it can produce** (cash
has none; a bond has GROSS + ACCRUED; OBL BRH adds BASE_INTEREST + INDEX_SPREAD). The
authoring UI reads this table to know an instrument type's `availableAmounts()`.

## Rules

- The `functionKey` must correspond to a function registered in code. Adding a row for
  an unregistered key is invalid — the registry is the single source both the engine
  and the UI read.
- Header-derived amounts (`QUANTITY`, `PRINCIPAL`, `ACCRUED`, `COUNTER`) are **never**
  registered here — they are computed generically from transaction inputs and belong to
  no instrument.
- Reuse without new code = a new `Instrument` row on an existing `InstrumentType` (its
  calculators already exist). A new `InstrumentType` that computes something genuinely
  new = register the function(s), then add the rows.
- **Authoring-time coherence** (also stated in
  [TransactionType](../transactions/TransactionType.md)): a **required**
  instrument-derived name in a transaction type's `declaredAmounts` must have an
  `InstrumentCalculator` row for **every** instrument type the type's eligibility
  permits; an **optional** name needs a row on at least one eligible type (on the
  others the amount is simply absent, e.g. WITHHOLDING on an exempt kind). Declaring
  `INDEX_SPREAD` as required on a type eligible for plain `BOND` is rejected at
  type-creation ("ordinary bonds do not compute INDEX_SPREAD"). The gap is caught when
  the type is defined, never when a transaction posts. The type-authoring UI enforces
  this by offering, for a given eligibility, only the union of the eligible types'
  `availableAmounts()`, plus the four header-derived names which are always available.
  Eligibility is selected on the same authoring screen, so every save of a type
  (creation or later change) re-runs the check. Deactivating a calculator row is
  rejected while an active type still declares its amount and is eligible for its
  instrument type.

## The registry contract

`functionKey` is a reference into a code-owned list, never free text. How the list is
built (annotations, framework hooks, reload handling) is a development decision; the
guarantees below are the contract:

1. **Code declares itself.** Each calculator function is declared in code with its key,
   the amount it computes, and the instrument types it is valid for. At startup the
   application collects these declarations into an in-memory registry: the single
   authoritative list of existing calculators. Nobody maintains it by hand; it is
   derived from the code, so it cannot go stale.
2. **The UI selects, never types.** When an `InstrumentCalculator` row is created,
   `functionKey` is chosen from the registry, filtered to the row's `instrumentTypeId`
   and `amountName`. A key not backed by code is not offerable, so typos and dangling
   references are structurally impossible.
3. **Startup validation seals it.** On save and on application startup, every stored
   `functionKey` is checked against the registry; a row pointing at a key the code no
   longer declares fails fast and loudly, never silently at posting time.
4. **Collisions fail at boot.** Two functions claiming the same
   `(instrumentType, amountName)` is a developer error and must stop startup, naming
   both functions, rather than silently overwriting.
5. **Built once, read-only, per process.** The registry is populated once at startup
   and never mutated while serving. Each worker rebuilds the identical registry from
   the same code: the same deterministic computation performed independently, not
   shared state that could diverge.

**Division of maintenance:**

| | Lives in | Maintained by |
|---|---|---|
| The catalog of *which calculators exist* | code (the registry, built from annotations) | developers, by writing and annotating functions |
| *Which* calculator an instrument type uses | the `InstrumentCalculator` table | admin, by selecting from the registry-backed dropdown |

The table stores a reference into a code-owned list — the data side can only ever
*select* a key, never *invent* one.

## Clean model

```
InstrumentCalculator
  id                uuid    PK
  instrumentTypeId  FK InstrumentType
  amountName        enum    -- instrument-derived names only (GROSS, WITHHOLDING,
                            --   INCOME, BASE_INTEREST, INDEX_SPREAD)
  functionKey       string  -- stable key of a registered code function; never a formula
  description       string?
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)
  unique (instrumentTypeId, amountName)
```

## Seeding logic

The concrete rows are decided with each instrument type part; the shape is:

- An instrument type carries rows only for the amounts its instruments produce: cash
  and provisions register nothing; a bond registers its coupon math (`GROSS`); OBL BRH
  registers its own `GROSS`, `BASE_INTEREST` and `INDEX_SPREAD`; equities and funds
  register their income math (`GROSS`).
- `WITHHOLDING` and `INCOME` are uniform across instrument types
  (`WITHHOLDING = GROSS × Instrument.withholdingRate`, `INCOME = GROSS − WITHHOLDING`).
  They are still registered as per-type rows pointing at shared functions rather than
  hardcoded as a generic step: this keeps the coherence rule uniform (one rule, no
  exemptions) at the cost of a few seed rows.
- OBL BRH deliberately has **no** `WITHHOLDING` row: it is exempt (its
  `withholdingRate` is NULL), so a type declaring WITHHOLDING as required cannot be
  authored against it, which is exactly right.
