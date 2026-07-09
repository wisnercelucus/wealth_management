# UserPreferences

A recipient's delivery preferences — **1:1 with a user**, keyed by the soft `userId`. The service
consults this row before routing a non-critical notification: it decides language, which channels
are allowed, which types are muted, and (with [QuietHours](QuietHours.md)) when to stay silent.
Absent a row, defaults apply and nothing is suppressed.

## Essential fields

| Field            | Type        | Req | Description                                                              |
| ---------------- | ----------- | --- | ----------------------------------------------------------------------- |
| `id`             | bigint      | ●   | Surrogate PK                                                            |
| `userId`         | string      | ●   | External user id — **soft reference** to the Party cluster, **unique** |
| `activeChannels` | json (list) | ●   | Enabled channels, e.g. `["IN_APP","EMAIL"]` (default in-app + email)   |
| `language`       | enum        | ●   | `Language`: `fr \| en` (default `fr`)                                  |
| `timezone`       | string      | ●   | IANA tz for quiet-hours math (default `America/Port-au-Prince`)        |
| `mutedTypes`     | json (list) | ●   | Free `type` strings the user silenced, matching `send()` (default `[]`)|
| + envelope       |             | ●   | `createdAt` / `updatedAt` — see [README](README.md)                   |

## Notes & rules

- **One row per user** — `userId` is unique; it is a soft string reference, no FK to auth/Party.
- **Suppression logic (non-critical only).** A [Delivery](Delivery.md) is `CANCELLED` at send time
  if the event `type ∈ mutedTypes`, its `channel ∉ activeChannels`, or `in_quiet_hours()` is true.
  Muting a `type` cancels every delivery of the event; a disabled channel or quiet hours cancels only
  the affected deliveries.
- **`CRITICAL` always bypasses** preferences, muting and quiet hours (whatever `type` the caller
  sets — e.g. an urgent alert).
- **`timezone`** is interpreted in IANA form via `zoneinfo`; it scopes the QuietHours windows to the
  user's local clock.
- **`language`** here is the default selector for template resolution and the notification's
  `language` field when the caller does not override it.

## Clean model

```
UserPreferences
  id             bigint  PK
  userId         string                 -- soft ref → Party/User, unique
  activeChannels json (list)            -- default ["IN_APP","EMAIL"]
  language       enum (fr | en)         -- default fr
  timezone       string                 -- default America/Port-au-Prince
  mutedTypes     json (list)            -- default []
  + envelope (createdAt, updatedAt)

  unique (userId)
  1 ─── * QuietHours
```
