# CurrencyPair

The **definition** of an FX pair base currency, quote currency, and thereby direction. This
is reference data; the **rate** on any given day is market data and lives in the market-data
layer

## Essential fields

| Field             | Type        | Req | Description                      |
| ----------------- | ----------- | --- | -------------------------------- |
| `id`              | uuid/bigint | ●   | Surrogate PK                     |
| `code`            | string(7)   | ●   | Pair code (`USDHTG`) — unique    |
| `baseCurrencyId`  | FK→Currency | ●   | Base currency (the 1 unit)       |
| `quoteCurrencyId` | FK→Currency | ●   | Quote currency (price of 1 base) |
| + envelope        |             | ●   | see README                       |

> **Direction** is the base→quote ordering itself (`USDHTG` = price of 1 USD in HTG) — no
> separate direction column needed.

## Clean model

```
CurrencyPair
  id               uuid    PK
  code             string  unique          -- e.g. "USDHTG"
  baseCurrencyId   FK Currency
  quoteCurrencyId  FK Currency
  + envelope
  unique (baseCurrencyId, quoteCurrencyId)
```
