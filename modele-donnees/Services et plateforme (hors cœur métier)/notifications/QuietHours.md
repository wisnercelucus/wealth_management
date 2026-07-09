# QuietHours

A silence window — a 1:many typed child of [UserPreferences](UserPreferences.md). A user may keep
several windows (e.g. nights and weekends), so windows are rows, not flat columns. During a window,
non-critical notifications are suppressed; `CRITICAL` always passes through.

## Essential fields

| Field         | Type                 | Req | Description                                                              |
| ------------- | -------------------- | --- | ----------------------------------------------------------------------- |
| `id`          | bigint               | ●   | Surrogate PK                                                            |
| `preferences` | FK→UserPreferences   | ●   | The owning preferences row (cascade delete)                            |
| `start`       | time                 | ●   | Window start (in the user's `timezone`)                                |
| `end`         | time                 | ●   | Window end; may wrap past midnight (e.g. `22:00 → 07:00`)              |
| `weekday`     | smallint             | ○   | `0..6` (Mon..Sun); `null` = applies every day                          |

## Notes & rules

- **Typed 1:many child** of `UserPreferences` (same shape idea as PartyContact / PartyAddress on the
  Party cluster) — cascades when the parent preferences are deleted.
- **Midnight wrap.** When `start > end` the window spans midnight: `contains(t)` matches
  `t ≥ start OR t ≤ end`. Otherwise it is the plain `start ≤ t ≤ end`.
- **Weekday scope.** `weekday = null` applies daily; a value `0..6` (Monday = 0) restricts the
  window to that day, evaluated in the parent's `timezone`.
- **No envelope.** This child has no `createdAt` / `updatedAt`; its lifecycle follows the parent.
- **Critical bypass.** Quiet hours never suppress `CRITICAL` notifications.

## Clean model

```
QuietHours
  id          bigint  PK
  preferences FK UserPreferences        -- cascade delete
  start       time
  end         time                      -- may wrap past midnight
  weekday     smallint?                 -- 0..6 (Mon..Sun); null = every day
  (no envelope — owned by parent)
```
