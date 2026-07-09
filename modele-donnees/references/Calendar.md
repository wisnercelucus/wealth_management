# Calendar

A named business-day calendar. ProFin needs the **Haiti** calendar (settlement, accrual, NAV
dates) and a **US** calendar for the Raymond James funds. Holiday dates hang off it in
[CalendarHoliday](CalendarHoliday.md).

## Essential fields

| Field         | Type        | Req | Description                                                      |
| ------------- | ----------- | --- | ---------------------------------------------------------------- |
| `id`          | uuid/bigint | ●   | Surrogate PK                                                     |
| `code`        | string(20)  | ●   | Calendar code (`HAITI`, `US`) — unique                           |
| `name`        | string      | ●   | Display name (`Haiti Business Calendar`)                         |
| `weekendDays` | string      | ○   | Non-working weekdays (`SAT,SUN`) — part of the business-day rule |
| `countryId`   | FK→Country  | ○   | Country the calendar belongs to                                  |
| + envelope    |             | ●   | see README                                                       |

## Behavior

The point of a calendar is the date math, which lives in code: `isBusinessDay(date)`,
`nextBusinessDay(date)`, `addBusinessDays(date, n)`, `rollDate(date, convention)` — keyed off
the weekend rule + the holiday rows, applying a [`BusinessDayConvention`](enums.md). This
service is what settlement (T+n), coupon/maturity rolling, and NAV dating all call.

## Clean model

```
Calendar
  id           uuid    PK
  code         string  unique
  name         string
  weekendDays  string?                -- e.g. "SAT,SUN" (or drop if always Sat/Sun)
  countryId    FK Country?
  + envelope
```
