# Analysis report — `profin/wealth_management/`

Structural and content review of the repository, conducted 2026-07-10.

---

## 1. What this repository is

It is a **documentation-only repository**: 66 Markdown files and one hand-authored SVG diagram,
4,906 lines in total. There is no source code, no build tooling, no tests, no dependency manifest.

It holds the **data-model specification and the operational workflow specification** for **ProFin**,
a Haitian wealth-management platform (regulator: the BRH, Banque de la République d'Haïti;
currencies HTG / USD / EUR; the current system is a migration away from a predecessor called **Axia**).

| Fact | Value |
| --- | --- |
| Git repository | Yes — its own repo, nested inside `profin/` |
| Remote | `git@github.com:wisnercelucus/wealth_management.git` |
| History | A single commit, `6aef6da` "initial commit", 2026-07-09, Wisner Celucus |
| Working tree | Clean, branch `master` |
| Files | 66 `.md` + 1 `.svg` |

Because the entire history is one commit, the real change history of the design lives inside
`modele-donnees/FOLLOW-UP.md` rather than in git.

---

## 2. Folder structure

```
wealth_management/
├── ProFin_Modele_Donnees.md          — top-level index: every planned module, one line each
├── modele-donnees/                   — the data model (58 files)
│   ├── FOLLOW-UP.md                  — the design changelog / decision log
│   ├── references/                   — shared kernel: Currency, Country, Calendar,
│   │                                   CalendarHoliday, CurrencyPair, enums
│   ├── Cœur investissement/          — the business core
│   │   ├── clients/                  — Party, roles, profile, contacts, addresses,
│   │   │                               documents, groups, BankAccount
│   │   ├── portefeuilles/            — Portfolio, PortfolioParty, PortfolioRole, PortfolioGroup
│   │   ├── instruments/              — Instrument, InstrumentType, AssetClass,
│   │   │   └── OBL BRH/                InstrumentCalculator, InstrumentGroup, OblBrh
│   │   ├── transactions/             — the ledger: Transaction, TransactionAmount,
│   │   │                               TransactionLeg, Lot, LotRelief + 5 config tables
│   │   ├── charges/                  — commissions, fees, TCA tax (configuration only)
│   │   ├── market-data/              — InstrumentPrice, FxRate
│   │   ├── position/                 — DRAFT-position-module.md  (active draft)
│   │   └── taxation/                 — DRAFT-charges-module.md   (superseded, see §6)
│   └── Services et plateforme (hors cœur métier)/
│       ├── maker-checker/            — generic approval gate
│       └── notifications/            — Notification, Delivery, Template, prefs, quiet hours
└── worflow/                          — [sic] operational process narratives
    └── instruments/obl brh/transactions/
        ├── README.md, flow.svg
        ├── client/    subscription, maturity, early-redemption
        └── house/     send-to-brh, brh-credits, debit-note, repatriation
```

Two document genres coexist. Under `modele-donnees/` the pages are **reference specifications**:
a purpose paragraph, an "Essential fields" table (`Field | Type | Req | Description`, with `●`
required / `○` optional), a "Notes & rules" section, a "Deliberately absent" section listing what is
intentionally *not* stored, and a closing "Clean model" pseudo-DDL block with inline `CHECK` and
`UNIQUE` constraints. Cluster READMEs add Mermaid ER diagrams. Under `worflow/` the pages are
**narrative process descriptions** with no schemas — sections such as *When it happens / What the
operator does / Transaction produced / Charges / Settlement lifecycle / Accounting / Control*.

Prose is written in **English throughout**; the domain vocabulary is deliberately kept in French
(*prime de succès*, *note de débit*, *rapatriement*, *cloisonnement*, *conseiller principal*), and
every catalog entity carries a `labelFr` / `labelEn` pair. Default UI language is `fr`, default
timezone `America/Port-au-Prince`.

---

## 3. The design, in one page

The model is built on a single organizing principle, stated in almost every file:
**what can be computed is never stored.** Positions, balances, market values, AUM, running accruals,
unrealized P&L, the OBL BRH baseline and its early-redemption penalty are all derived at read time.
A stored figure that could be recomputed is called a "banned derivable" and is rejected by design.

The ledger is a five-layer chain, and everything downstream reads it:

1. **Transaction** — the event header. It carries exactly five deal-term inputs (`quantity`, `price`,
   `accrued`, `fxRate`, `settlementCashInstrumentId`) plus three address columns. Status is
   `PENDING | POSTED | REJECTED`; once posted a row is immutable and can only be reversed, never
   edited or deleted. A `postingSequence` bigint gives a monotonic "knowledge order" that FIFO
   sorts on, so backdated entries never disturb walks that already ran.
2. **TransactionAmount** — the computed *witnesses*, one row per named amount, frozen at posting.
   The name set is closed at **nine**: `QUANTITY`, `PRINCIPAL`, `ACCRUED`, `COUNTER`, `GROSS`,
   `WITHHOLDING`, `INCOME`, `BASE_INTEREST`, `INDEX_SPREAD`. The first four are derived from the
   header by a generic engine; the last five are delegated to the instrument type through the
   **InstrumentCalculator** registry, which maps `(instrumentTypeId, amountName)` to a code-registered
   `functionKey` (never a formula string; collisions fail at boot).
3. **TransactionLeg** — the financial truth: one signed movement, sign carries direction, `amount != 0`
   (an inapplicable conditional leg is omitted, never zeroed). Each leg stamps a `lotEffect` from a
   closed set of five — `ESTABLISH`, `RELIEVE_REALIZE`, `CARRY`, `ADJUST_BASIS` (dormant in MVP),
   `RESTATE`. Legs are generated from the transaction type's **contract lines**
   (`TransactionTypeLeg`), whose `portfolioMode` resolves the destination book through one of
   **seven** modes (`TRANSACTION`, `TRANSACTION_CASH`, `TRANSACTION_BANK`, `TRANSACTION_COUNTERPARTY`,
   `TRANSACTION_COUNTERPARTY_CASH`, `TRANSACTION_INSTRUMENT_ISSUER`, `SPECIFIC`).
4. **Lot / LotRelief** — the cost memory and the realized P&L of record. Lots keep the full acquisition
   history forever; the cost-basis method is a *walk strategy, not a data shape*, so FIFO is the only
   MVP implementation and `AVERAGE_COST` or LIFO become future code with no schema change. Realized
   P&L is split into a price component and an FX component and frozen. Reversal writes offsetting
   rows; nothing is ever deleted.
5. **Position** (still a draft) — a cache over legs and lots that stores **quantity only**, updated or
   marked `isStale` in the same database transaction as the ledger write. The draft explicitly cites
   this as making "the Axia 12-day desync bug" structurally impossible.

Around that core:

- **Transaction types are pure declaration.** They own no behavior — no `behaviorMode`, no posting
  function key. Legs are data; posting is one generic engine. Changed economics means a new type code,
  because a type with posted transactions is frozen.
- **Charges own no instance tables.** A posted charge *is* its movement: a `TransactionLeg` stamped with
  a nullable `chargeNameId`. The charges module is configuration only — a charge-name catalog (open,
  admin-creatable, unlike the closed nine witness names), plus policies made of a reusable condition
  (*on whom*) and a reusable effect (*how much*). Tax follows the service (`COMMISSION → TCA` via
  `taxChargeNameId`), and there is no tax on tax.
- **Identity is separated from relationship.** `Party` is a lean identity row; roles hang off
  `PartyRoleMembership`, and portfolio roles live on the `PortfolioParty` link, so `Portfolio` has no
  `clientId` at all. AUM is driven by `Portfolio.bookKind = CLIENT` intersected with
  `AssetClass.isAumEligible`, not by party role — which is how fund books avoid double-counting.
- **Maker-checker is entity-agnostic.** It approves an opaque `payload` against a `targetType` string,
  never a foreign key. A flow is gated if and only if an active policy exists for its
  `(targetType, operation)` — turning approval on or off is data, not code. A change has no effect until
  it reaches `APPLIED`, and approval plus application happen in one database transaction.
- **Notifications are a generic delivery engine.** One `send()` produces one `Notification` and one
  `Delivery` per channel (`IN_APP | EMAIL | SMS`); rendered title and body are frozen at creation so
  template edits never rewrite history. `CRITICAL` priority bypasses mutes, channel preferences and
  quiet hours. The module is append-only, with no soft-delete, for the BRH's 10-year retention rule.

**OBL BRH** is the one instrument modeled end to end: a BRH bond with no coupons and no secondary
market, paying in full at maturity, with an index-protection spread applied only at maturity, and a
*prime de succès* that BRH reclaims on early redemption (the client sees it as a penalty; it is
explicitly not a charge). It is modeled in three tiers — Template → Emission → Subscription — and its
money flow is what the entire `worflow/` tree documents, across a client ledger (subscription,
maturity, early redemption) and a house ledger (send-to-brh, debit note, brh credits, repatriation),
tied together by a settlement lifecycle `pending → sent → debited → credited`.

---

## 4. Maturity of the documentation

The two subsystems specified in depth — the **transaction ledger** and the **instrument/party/portfolio
clusters** — read as settled, near-final specifications. They are internally consistent, they state their
invariants formally, and `FOLLOW-UP.md` shows the design being actively pressure-tested: a nine-witness
BRH maturity was pushed through the model as a stress test, which produced three definition fixes; an
entire portfolio-resolution mode (`TRANSACTION_COUNTERPARTY_BANK`) was dropped after the owner showed it
was degenerate; the charges module was rebuilt from three drafts into a configuration-only design.

`FOLLOW-UP.md` is the single most valuable file in the repository. It is a dated decision log covering
2026-07-03 → 2026-07-08, in which each entry states the decision, the reasoning, and the exact files it
was applied to. It closes with an "Open decisions" section holding one item and a "Deferred by decision"
section holding two.

---

## 5. Coverage gap: the index describes far more than exists

`ProFin_Modele_Donnees.md` presents an inventory of 22 modules. Ten are documented; twelve are named
only as a one-line intent.

**Documented:** `references`, `clients`, `portefeuilles`, `instruments`, `transactions`, `market-data`,
`charges`, `position` (draft), `maker-checker`, `notifications`.

**Named but not written:** `tiers-et-comptes`, `taxation`, `comptabilite`, `actions-corporatives`,
`fonds`, `pension`, `securite-acces`, `am-services-ia`, `taches-batch`, `documents`, `audit`,
`integration`, `profin-online`, `reporting`.

Within the instrument cluster, the type-specific parts for `BOND`, `EQUITY` and `FUND` are all marked
"part to be designed" — only `OBL_BRH` and `CASH` are complete. Several shared enums in
`references/enums.md` (`BusinessDayConvention`, `DayCountConvention`, `PeriodFrequency`) are labeled
"proposed set" rather than definitive.

---

## 6. Findings

**The top-level index is out of date.** `ProFin_Modele_Donnees.md` lists a `frais` module, but the
folder that exists is `charges/`. It lists a `taxation` module, but taxation was folded into charges
and the taxation folder now holds only a dead draft. It does not list `position/` at all, even though
that module is under active design. The index also folds `tiers-et-comptes` (Émetteur, CompteBancaire)
into its own module, whereas `BankAccount` actually lives in `clients/` and issuers are ordinary
parties. Anyone reading the index first will form a wrong map of the repository.

**A superseded draft is still on disk.** `FOLLOW-UP.md` records
`taxation/DRAFT-charges-module.md` as "deleted — superseded by the new charges/ module docs." The file
still exists (277 lines). Its own header says "DRAFT — SUPERSEDED 2026-07-08 … This file can be
deleted." It describes a design that was explicitly abandoned — instance tables
(`TransactionChargeAmount`), a `destinationPortfolioId` routing column, two legs per charge — all of
which the final design rejects. Read as current, it is actively misleading. Either delete the file or
correct the changelog entry.

**The workflow folder is misspelled.** `worflow/` should be `workflow/`. The typo is consistent
throughout the tree's internal relative links, so a rename requires updating those references too.

**One open decision is still unresolved.** `FOLLOW-UP.md` flags the `CASH_DEPOSIT` worked example in
`TransactionTypeLeg.md`: the example shows both legs signed `+`, while the owner describes the deposit
as credit-client / debit-issuer-side. The sign convention for the house-side leg is awaiting a ruling.

**Two parked design items remain.** `TransactionAmount.md` parks a candidate tenth witness name for
third-party fees netted at source by Raymond James, pending confirmation of the modalities.
`Lot.md` and `TransactionLeg.md` define `ADJUST_BASIS` but mark it dormant for the MVP.

**Documented cross-references point outside their clusters.** `InstrumentCalculator.md`,
`InstrumentType.md`, `InstrumentPrice.md` and the market-data README link to
`../transactions/README.md` and siblings. These resolve correctly on disk, but they mean the instrument
and market-data clusters cannot be read standalone — worth knowing before anyone extracts a module.

**A recurring dependency inversion is worth calling out as a strength, not a defect.** Accounting binds
*its* tables to transaction type codes, rather than transactions carrying a `journalTemplateId`. Charge
policies are reconstructed from validity dates rather than stamped on posted rows. Maker-checker knows
nothing about the entities it approves. In each case the dependency arrow points away from the ledger,
which is what keeps the ledger's posted rows frozen and small.

---

## 7. Suggested next steps

1. Delete `taxation/DRAFT-charges-module.md`, or amend the `FOLLOW-UP.md` entry that claims it is gone.
2. Regenerate the module tree and the module list in `ProFin_Modele_Donnees.md` from what actually
   exists, and mark the unwritten modules as planned rather than describing them in the present tense.
3. Rename `worflow/` to `workflow/` and fix the internal links in the same commit.
4. Get the owner's ruling on the `CASH_DEPOSIT` leg signs, then close the open-decisions section.
5. Promote `position/DRAFT-position-module.md` once approved, and apply the "Impacts on other docs"
   checklist it carries (the transactions README rewording and a `FOLLOW-UP.md` entry).
6. Consider committing more often. A design this carefully argued deserves a git history that shows the
   argument; today the whole thing is one commit and `FOLLOW-UP.md` is carrying that load alone.
