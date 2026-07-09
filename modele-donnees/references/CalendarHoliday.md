# CalendarHoliday

The holiday dates for a [Calendar](Calendar.md). Explicit dated rows — one per holiday per
calendar — not recurrence rules.

## Essential fields

| Field        | Type        | Req | Description                                                                  |
| ------------ | ----------- | --- | ---------------------------------------------------------------------------- |
| `id`         | uuid/bigint | ●   | Surrogate PK                                                                 |
| `calendarId` | FK→Calendar | ●   | The calendar this holiday belongs to                                         |
| `date`       | date        | ●   | The non-business date                                                        |
| `name`       | string      | ○   | Holiday label (`Jour de l'Indépendance`) — display only                      |
| `repeat`     | bool        | ○   | decide wether tis holiday repeat every year or is only for the specific year |
| + envelope   |             | ●   | see README                                                                   |

## Notes & rules

- One row per `(calendarId, date)` — unique.

## Clean model

```
CalendarHoliday
  id          uuid    PK
  calendarId  FK Calendar
  date        date
  name        string?
  repeat      bool
  + envelope
  unique (calendarId, date)
```
