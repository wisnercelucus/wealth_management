# PartyAddress

A party's postal addresses a 1:many typed child of [Party](Party.md). A party (especially a
pension employer or employee) carries several, by type: Domicile, Bureau, Correspondance.
`isPrimary` marks the default.

## Essential fields

| Field        | Type        | Req | Description                                                              |
| ------------ | ----------- | --- | ------------------------------------------------------------------------ |
| `id`         | uuid/bigint | ●   | Surrogate PK                                                             |
| `partyId`    | FK→Party    | ●   | The party                                                                |
| `type`       | enum        | ●   | `AddressType`: `DOMICILE \| BUREAU \| CORRESPONDANCE` (extend as needed) |
| `line1`      | string      | ●   | Address line 1                                                           |
| `line2`      | string      | ○   | Line 2                                                                   |
| `city`       | string      | ●   | City / commune                                                           |
| `region`     | string      | ○   | Department / state                                                       |
| `postalCode` | string      | ○   | Postal code                                                              |
| `countryId`  | FK→Country  | ●   | Country                                                                  |
| `isPrimary`  | bool        | ●   | The default address                                                      |
| + envelope   |             | ●   | see README                                                               |

## Notes & rules

- Fields are inline (no shared `Address` entity) each party owns its addresses.
- One primary per `(partyId)` the default for statements / correspondence.
- `AddressType` is a local enum: `DOMICILE | BUREAU | CORRESPONDANCE`.

## Clean model

```
PartyAddress
  id          uuid    PK
  partyId     FK Party
  type        enum (DOMICILE | BUREAU | CORRESPONDANCE)
  line1       string
  line2       string?
  city        string
  region      string?
  postalCode  string?
  countryId   FK Country
  isPrimary   bool
  + envelope
```
