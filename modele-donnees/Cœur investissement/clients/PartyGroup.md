# PartyGroup

A **generic logical grouping** of parties — the buckets are arbitrary and can mean whatever the
business needs (cohorts, lists, tags). **Not** family / related-party structure: there is no
directed relationship here, just membership. Distinct from `PortfolioGroup`, which groups
_portfolios_ by segment / account-type. Many-to-many via a membership link.

## PartyGroup

| Field      | Type        | Req | Description         |
| ---------- | ----------- | --- | ------------------- |
| `id`       | uuid/bigint | ●   | Surrogate PK        |
| `code`     | string(50)  | ●   | Group code : unique |
| `name`     | string      | ●   | Group name          |
| + envelope |             | ●   | see README          |

## PartyGroupMembership the link

| Field          | Type          | Req | Description      |
| -------------- | ------------- | --- | ---------------- |
| `id`           | uuid/bigint   | ●   | Surrogate PK     |
| `partyGroupId` | FK→PartyGroup | ●   | The group        |
| `partyId`      | FK→Party      | ●   | The member party |
| + envelope     |               | ●   | see README       |

## Notes & rules

- **Not `PortfolioGroup`** different grain, different FK. PortfolioGroup buckets portfolios;
  PartyGroup buckets parties.
- Membership carries **no** family / relationship semantics — it is just an arbitrary grouping.

## Clean model

```
PartyGroup
  id    uuid    PK
  code  string  unique
  name  string
  + envelope

PartyGroupMembership
  id            uuid   PK
  partyGroupId  FK PartyGroup
  partyId       FK Party
  + envelope
  unique (partyGroupId, partyId)
```
