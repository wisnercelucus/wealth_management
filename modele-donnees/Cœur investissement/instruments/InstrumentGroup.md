# InstrumentGroup

**Categorization buckets** for instruments — the instrument-side scope handle for
charge and taxation rules. Distinct from the kind ([InstrumentType](./InstrumentType.md):
*what an instrument is*) and the asset class ([AssetClass](./AssetClass.md): *how it
behaves* for reporting and AUM): a group is a free commercial bucket ("OBL BRH",
"Fonds NAOS", "Money market USD") an admin curates for pricing rules.

> Assumes the shared envelope (see [README](./README.md)).

## InstrumentGroup fields

| Field      | Type       | Req | Description                                                  |
| ---------- | ---------- | --- | ------------------------------------------------------------ |
| `id`       | uuid       | ●   | Surrogate primary key.                                       |
| `code`     | string(50) | ●   | Group code. Join on `id`, not this.                          |
| `name`     | string     | ●   | Group name.                                                  |
| + envelope |            | ●   | See [README](./README.md).                                   |

## InstrumentGroupMembership — the link

| Field               | Type               | Req | Description                                                  |
| ------------------- | ------------------ | --- | ------------------------------------------------------------ |
| `id`                | uuid               | ●   | Surrogate primary key.                                       |
| `instrumentGroupId` | FK→InstrumentGroup | ●   | The group.                                                   |
| `instrumentId`      | FK→Instrument      | ●   | The member instrument.                                       |
| + envelope          |                    | ●   | See [README](./README.md).                                   |

## Notes & rules

- An instrument can be in **many** groups → membership is its own table, never a column
  on either side.
- One active membership per `(instrumentId, instrumentGroupId)`.
- Membership is assigned **explicitly**: no dynamic, criteria-based group rules. Adding
  a new emission to a charged group is part of creating the instrument. A rule that must
  catch every instrument of a kind automatically should scope on the kind
  (InstrumentType), not on a group.

## Clean model

```
InstrumentGroup
  id          uuid    PK
  code        string  unique
  name        string
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)

InstrumentGroupMembership
  id                 uuid   PK
  instrumentGroupId  FK InstrumentGroup
  instrumentId       FK Instrument
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)
  unique (instrumentId, instrumentGroupId) where isActive
```
