# PartyDocument

A party's identity documents a 1:many typed child of [Party](Party.md). A party may hold more
than one , so documents are rows, not flat columns. The number is **PII**.

## Essential fields

| Field            | Type          | Req | Description                                                                                            |
| ---------------- | ------------- | --- | ------------------------------------------------------------------------------------------------------ |
| `id`             | uuid/bigint   | ●   | Surrogate PK                                                                                           |
| `partyId`        | FK→Party      | ●   | The party                                                                                              |
| `type`           | enum          | ●   | `IdDocumentType`: `NATIONAL_ID \| PASSPORT \| DRIVERS_LICENSE \| TAX_ID \| BIRTH_CERTIFICATE \| OTHER` |
| `number`         | string        | ●   | Document number **PII** (encrypt at rest)                                                              |
| `expiryDate`     | date          | ○   | Expiry date                                                                                            |
| `issueDate`      | date          | ○   | Issue date                                                                                             |
| `issuerCountryId`| FK→Country    | ○   | Issuing country                                                                                        |
| `isPrimary`      | bool          | ●   | The primary identity document                                                                          |
| + envelope       |               | ●   | see README                                                                                             |

## Notes & rules

- Typed 1:many child (no flat `id` / `id2` columns) — same shape as PartyContact / PartyAddress.
- **`number` is PII** — encrypt at rest, restrict read access, keep out of logs.
- `IdDocumentType` is a local enum: `NATIONAL_ID | PASSPORT | DRIVERS_LICENSE | TAX_ID | BIRTH_CERTIFICATE | OTHER`.
  (Canonicalize messy source labels — driver's-licence/national-ID/passport variants, typos, junk —
  onto these six at migration.)
- One primary per `(partyId)`.

## Clean model

```
PartyDocument
  id          uuid    PK
  partyId     FK Party
  type        enum (NATIONAL_ID | PASSPORT | DRIVERS_LICENSE | TAX_ID | BIRTH_CERTIFICATE | OTHER)
  number      string                 -- PII, encrypt at rest
  expiryDate  date?
  issueDate   date?
  issuerCountryId FK Country
  isPrimary   bool
  + envelope
```
