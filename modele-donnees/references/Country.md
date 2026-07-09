# Country

ISO country list — referenced by addresses, issuer domicile, and (in KYC) nationality. Small,
static, broadly referenced. Load the ISO 3166 set.

## Essential fields

| Field      | Type        | Req | Description                                    |
| ---------- | ----------- | --- | ---------------------------------------------- |
| `id`       | uuid/bigint | ●   | Surrogate PK                                   |
| `code`     | string(2)   | ●   | ISO 3166-1 alpha-2 (`HT`, `US`, `FR`) — unique |
| `name`     | string      | ●   | Display name (`Haiti`)                         |
| + envelope |             | ●   | see README                                     |

## Clean model

```
Country
  id    uuid    PK
  code  string  unique             -- ISO 3166-1 alpha-2
  name  string
  + envelope
```
