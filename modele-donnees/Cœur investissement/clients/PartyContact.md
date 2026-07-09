# PartyContact

A party's emails and phones a 1:many typed child of [Party](Party.md). A party carries
several: a work email and a personal one, a mobile and an office line. `isPrimary` marks the one
to use (e.g. the notification target).

## Essential fields

| Field       | Type        | Req | Description                                          |
| ----------- | ----------- | --- | ---------------------------------------------------- |
| `id`        | uuid/bigint | ●   | Surrogate PK                                         |
| `partyId`   | FK→Party    | ●   | The party                                            |
| `type`      | enum        | ●   | `ContactMethod`: `EMAIL \| MOBILE \| PHONE \| FAX` (extend as needed) |
| `value`     | string      | ●   | The address / number                                 |
| `isPrimary` | bool        | ●   | The preferred contact of this type                   |
| + envelope  |             | ●   | see README                                           |

## Notes & rules

- The portal's login / 2FA email is **separate** — it lives in the portal account, not here.
  PartyContact is the business contact record.
- One primary per `(partyId, type)` — the row the platform reaches for first.
- `ContactMethod` is a local enum: `EMAIL | MOBILE | PHONE | FAX`.

## Clean model

```
PartyContact
  id         uuid   PK
  partyId    FK Party
  type       enum ContactMethod (EMAIL | MOBILE | PHONE | FAX)
  value      string
  isPrimary  bool
  + envelope
```
