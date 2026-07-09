# PartyRole

The **catalog of roles** a party can hold — a small, seeded reference list. Assigning a role to a
party is [PartyRoleMembership](PartyRoleMembership.md).

> Role *codes* drive code branching (a `CLIENT` behaves differently from an `EMETTEUR`), so this
> catalog is seeded and stable — new roles come with code, not just data.
> Assumes the shared envelope (see [README](README.md)).

## Essential fields

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid/bigint | ● | Surrogate PK |
| `code` | string(40) | ● | Role code — unique |
| `name` | string | ● | Display name |
| + envelope | | ● | see README |

## The roles (seeded)

| code | meaning |
|------|---------|
| `CLIENT` | An investing client |
| `CONSEILLER` | Investment advisor on the relationship (portfolio-scoped in practice) |
| `EMETTEUR` | Issuer of a security (includes the funds ProFin distributes) |
| `BANQUE` | A bank ProFin settles through / its counterparty |
| `GESTIONNAIRE_FONDS` | External fund manager (e.g. the foreign-fund partner) |
| `VALORISATION` | External valuation company |
| `EMPLOYEUR` | A pension-plan sponsoring employer |
| `EMPLOYE` | A pension-plan member |

## Clean model

```
PartyRole
  id    uuid    PK
  code  string  unique
  name  string
  + envelope
```
