# PortfolioRole

The **catalog of roles** a party can play on a portfolio. Referenced by
[PortfolioParty](PortfolioParty.md). The role is the **single source of truth** for whether
a party is an owner, beneficiary, advisor, etc. its `name` conveys the category, so there
is no separate category field.

## Essential fields

| Field                   | Type       | Req | Description                                                                                                 |
| ----------------------- | ---------- | --- | ----------------------------------------------------------------------------------------------------------- |
| `id`                    | uuid       | ●   | Surrogate primary key.                                                                                      |
| `code`                  | string(50) | ●   | Stable role code. Application logic and reports key off it — keep it stable.                                |
| `name`                  | string     | ●   | Role name, self-categorizing (e.g. "Beneficial Owner").                                                     |
| `description`           | string     | ○   | Longer explanation of the role.                                                                             |
| `canViewInClientPortal` | bool       | ●   | Whether a holder of this role may view the portfolio in the **end-client portal** (ProFin Online).          |
| + envelope              |            | ●   | See [README](README.md): `isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy`. |

## Roles to seed

| code    | name                                               |
| ------- | -------------------------------------------------- |
| `18`    | Financial Advisor (Conseiller principal)           |
| `CP2`   | Secondary Advisor (Conseiller 2) — new ProFin code |
| `AGENT` | Agent — new ProFin code                            |
| `10`    | Beneficial Owner                                   |
| `5`     | Account Administrator                              |
| `33`    | Primary Beneficiary                                |
| `34`    | Secondary Beneficiary                              |
| `52`    | Tertiary Beneficiary                               |
| `53`    | Quaternary Beneficiary                             |
| `40`    | Authorized Signatory                               |
| `50`    | Legal Representative                               |
| `51`    | Power of Attorney                                  |
| `47`    | Shareholder                                        |
| `11`    | President                                          |

## Notes & rules

- **Role ≠ party type.** What a party _is_ (client / issuer / custodian) lives on the Party
  record; what it _does on this portfolio_ is the role.
- **Portal visibility vs staff permissions.** `canViewInClientPortal` governs only what a party
  holding this role sees in the **end-client portal**. Internal staff permissions (who in ProFin may
  do what) stay in the RBAC / security layer — a portfolio role is a client-side relationship, not a
  staff permission set. Set the flag per role in the seed (e.g. owners / beneficial owners = true).

## Clean model

```
PortfolioRole
  id                    uuid    PK
  code                  string  unique
  name                  string
  description           string?
  canViewInClientPortal bool
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)
```
