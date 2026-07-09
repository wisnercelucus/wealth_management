# PortfolioGroup

**Categorization buckets** for portfolios

## PortfolioGroup fields

| Field      | Type       | Req | Description                                                                                                 |
| ---------- | ---------- | --- | ----------------------------------------------------------------------------------------------------------- |
| `id`       | uuid       | ●   | Surrogate primary key.                                                                                      |
| `code`     | string(50) | ●   | Group code.                                                                                                 |
| `name`     | string     | ●   | Group name (e.g. "Compte Mineur", "Pension", "Institution").                                                |
| + envelope |            | ●   | See [README](README.md): `isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy`. |

## PortfolioGroupMembership the link

| Field              | Type              | Req | Description                                                                                                 |
| ------------------ | ----------------- | --- | ----------------------------------------------------------------------------------------------------------- |
| `id`               | uuid              | ●   | Surrogate primary key.                                                                                      |
| `portfolioGroupId` | FK→PortfolioGroup | ●   | The group.                                                                                                  |
| `portfolioId`      | FK→Portfolio      | ●   | The member portfolio.                                                                                       |
| + envelope         |                   | ●   | See [README](README.md): `isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy`. |

## Groups list

Compte - Retail 1, Compte - Retail 2, Compte - Corporate 1 / 2 / 3, Compte Mineur,
Institution, Clients - Services Financiers, Banking / Deposit Relationship, Compte - FOREX,
Gestion de Trésorerie, Pension, Group Pension Plan, Individual Pension Plan,
In-House Mutual Funds, Actionnaire ProFin, Collaborateur ProFin, Fournisseur - Vendeur,
Brokers, Offre Privée d'Investissement, Unallocated.

## Notes & rules

- A portfolio can be in **many** groups → membership is its own table, never a column on
  either side.
- One active membership per `(portfolioId, portfolioGroupId)`.
- Membership is assigned explicitly no dynamic, criteria-based group rules.

## Clean model

```
PortfolioGroup
  id          uuid    PK
  code        string  unique
  name        string
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)

PortfolioGroupMembership
  id                uuid   PK
  portfolioGroupId  FK PortfolioGroup
  portfolioId       FK Portfolio
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)
  unique (portfolioId, portfolioGroupId) where isActive
```
