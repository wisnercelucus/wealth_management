# PartyRoleMembership

The **M:N** assignment of [roles](PartyRole.md) to a [Party](Party.md). A party can hold several —
a company can be `CLIENT` + `EMETTEUR`; an external manager is `EMETTEUR` + `GESTIONNAIRE_FONDS`.
One row per role held.

## Essential fields

| Field         | Type         | Req | Description   |
| ------------- | ------------ | --- | ------------- |
| `id`          | uuid/bigint  | ●   | Surrogate PK  |
| `partyId`     | FK→Party     | ●   | The party     |
| `partyRoleId` | FK→PartyRole | ●   | The role held |
| + envelope    |              | ●   | see README    |

## Notes & rules

- One active row per `(partyId, partyRoleId)`.

## Clean model

```
PartyRoleMembership
  id           uuid   PK
  partyId      FK Party
  partyRoleId  FK PartyRole
  + envelope
  unique (partyId, partyRoleId)
```
