# Currency

The currencies ProFin operates in — HTG, USD, EUR. Referenced platform-wide (portfolios,
instruments, prices, accounts, transactions): the most depended-on table in the layer. Load the
ISO 4217 set; only a handful are in active use.

## Essential fields

| Field      | Type        | Req | Description                                                |
| ---------- | ----------- | --- | ---------------------------------------------------------- |
| `id`       | uuid/bigint | ●   | Surrogate PK                                               |
| `code`     | string(3)   | ●   | ISO 4217 alpha (`HTG`, `USD`, `EUR`) — unique business key |
| `name`     | string      | ●   | Display name (`Haitian Gourde`)                            |
| `symbol`   | string(5)   | ●   | Display symbol (`G`, `$`, `€`) — statements and UI         |
| `decimals` | int         | ●   | Minor-unit digits for rounding/display (HTG/USD = 2)       |
| `kind`     | enum        | ●   | Domestic/Foreign                                           |
| + envelope |             | ●   | see README                                                 |

## Not included

- **No rate / FX column** — a rate is market data, not an attribute of a currency. The _pair_
  is [CurrencyPair](CurrencyPair.md); the _value_ is the market-data layer.

## Clean model

```
Currency
  id        uuid    PK
  code      string  unique          -- ISO 4217 alpha
  name      string
  symbol    string
  decimals  int
  kind      enum
  + envelope
```
