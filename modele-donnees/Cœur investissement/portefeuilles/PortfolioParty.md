# PortfolioParty

The **link** between a portfolio and a party (client, advisor, beneficiary…), carrying the
role that party plays on that account. Many-to-many: a portfolio has many parties; a party
holds many portfolios.

## Essential fields

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `portfolioId` | FK→Portfolio | ● | The portfolio. |
| `partyId` | FK→Party | ● | The related party. |
| `portfolioRoleId` | FK→PortfolioRole | ● | The role this party plays here. **Single source of truth** for whether the party is an owner, beneficiary, advisor, etc. |
| `ownershipPerc` | decimal(9,4) | ○ | Ownership %. Informational (joint accounts); does **not** drive entitlement or tax splits. |
| `effectiveDate` | date | ● | When the relationship took effect. |
| `endDate` | date | ○ | When the relationship ended; `NULL` while active. Keeps closed links for history. |
| + envelope | | ● | See [README](README.md): `isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy`. |

## Design notes

- **The role lives on the link, not on the party.** The same person can be an Owner on one
  portfolio and an Advisor on another. See [PortfolioRole](PortfolioRole.md).
- **No `isOwner` / `isBeneficiary` flags.** They would restate `portfolioRoleId` and could
  drift from it — derive "is owner / is beneficiary" from the role.
- **UBO** is captured as the **Beneficial Owner** role, not a dedicated field. The deeper
  "natural person behind a party" beneficial-ownership chain is deferred.
- The **advisor (CP)** relationship is this link with an advisor role — **Conseiller principal**,
  **Conseiller 2**, or **Agent** (see [PortfolioRole](PortfolioRole.md)) — with `isActive = true`.
  The client's CP is derived by aggregating these links across the client's portfolios (there is no
  client-level advisor field).
- Joint ownership: several active links on the same portfolio.

## Rules

- One active row per `(portfolioId, partyId, portfolioRoleId)`.
- `endDate` is stamped when the link is closed; the row is retained for history (don't hard-delete).

## Clean model

```
PortfolioParty
  id               uuid          PK
  portfolioId      FK Portfolio
  partyId          FK Party
  portfolioRoleId  FK PortfolioRole
  ownershipPerc    decimal(9,4)?   -- informational
  effectiveDate    date
  endDate          date?
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)
  unique (portfolioId, partyId, portfolioRoleId) where isActive
```
