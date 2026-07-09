# Notification

The permanent record of a single logical **event** to notify — "Jean's margin call", "dividend
reminder for portfolio X". **One row per event**, whatever the number of channels that carry it. The
physical sends live in [Delivery](Delivery.md) rows (1:many): the FK there is what correlates the
email and the SMS of the same event. A Notification holds the *meaning* (`type`, `priority`,
`metadata`) once; the *outcome* (status, retries, timestamps) is tracked per channel on its
deliveries. Append-only (BRH 10-year audit) — an event is retired by cancelling its deliveries,
never deleted.

## Essential fields

| Field          | Type          | Req | Description                                                                       |
| -------------- | ------------- | --- | -------------------------------------------------------------------------------- |
| `id`           | uuid          | ●   | Surrogate PK (UUID4)                                                             |
| `recipientId`  | string        | ●   | External user id — **soft reference** to the Party cluster (no FK), indexed     |
| `dedupKey`     | string        | ○   | Optional idempotency key from the caller (e.g. `transactions:<uuid>:ALERT`), unique |
| `type`         | string (free) | ●   | Free type set by the caller (e.g. `OPERATION`, `ALERT`); engine never constrains it |
| `priority`     | enum          | ●   | `Priority`: `LOW \| NORMAL \| HIGH \| CRITICAL` (default `NORMAL`)              |
| `language`     | enum          | ●   | `Language`: `fr \| en` — resolved once for the event (default from preferences) |
| `metadata`     | json          | ●   | Free-form context **and** the template render variables; defaults to `{}`       |
| `sourceType`   | string        | ○   | Source entity type, `app.Model` form (e.g. `"transactions.Transaction"`) — soft ref, no FK |
| `sourceId`     | string        | ○   | Id of the source row (e.g. the transaction UUID)                                |
| `scheduledAt`  | datetime      | ○   | Deferred send time for the whole event; `null` = immediate                      |
| + envelope     |               | ●   | `createdAt` / `updatedAt` — see [README](README.md)                            |

## Notes & rules

- **One event, many deliveries.** A margin call sent by EMAIL *and* SMS is **one** `Notification`
  with **two** [Delivery](Delivery.md) rows. Never store a channel on the Notification — the channel,
  its rendered content and its status all live on the delivery.
- **`recipientId` is a soft reference** — a plain string, no DB FK. Keeps the module decoupled from
  the auth/Party cluster; resolve the real user in the consuming layer.
- **`dedupKey` gives exactly-once at the event level.** When set, the service skips creating a new
  event (and its deliveries) if one already exists for the same key — so a replayed batch or a double
  call does not duplicate. Optional: leave `null` when the guarantee isn't needed.
- **No rendered content here.** `title` / `body` are per-channel (SMS ≠ email) and live on
  [Delivery](Delivery.md). The Notification keeps the raw `metadata` so any delivery can be rendered
  and audited.
- **`metadata` has a double role:** (a) audit context, and (b) the **render variables** for the
  template — the keys the caller supplies (`ticker`, `montant`…) are exactly `{{ ticker }}`,
  `{{ montant }}` in [NotificationTemplate](NotificationTemplate.md).
- **`sourceType` + `sourceId` is a polymorphic reference.** `sourceType` names the entity type in
  `app.Model` form (a module name alone would be ambiguous), `sourceId` the row. Both are plain
  strings — **no FK / GenericForeignKey** — so the module stays decoupled; the consuming layer
  resolves them (e.g. a deep link "open the transaction").
- **`language` is resolved once** from `UserPreferences.language` (or the caller's override) and
  applies to every delivery of the event.
- **Deferred sends** (`scheduledAt` in the future): the event and its deliveries stay `PENDING`; the
  `process_pending_notifications` beat task routes them once due. A past `scheduledAt` sends now.
- **Suppression is per delivery** — muting a `type` cancels all deliveries; disabling a channel or
  quiet hours cancels only the affected ones. `CRITICAL` bypasses all suppression. See
  [Delivery](Delivery.md) and [UserPreferences](UserPreferences.md).
- **PII / confidentiality.** `metadata` may carry client financial data; keep out of logs, restrict
  read access. Retained 10 years (BRH).

## Clean model

```
Notification
  id            uuid    PK
  recipientId   string                 -- soft ref → Party/User, indexed
  dedupKey      string?                -- optional idempotency key, unique
  type          string  -- free; e.g. ALERT | OPERATION | CONTENT | SYSTEM (caller-defined)
  priority      enum (LOW | NORMAL | HIGH | CRITICAL)
  language      enum (fr | en)
  metadata      json                   -- free-form context + template render vars, default {}
  sourceType    string?                -- "app.Model", e.g. transactions.Transaction (soft ref, no FK)
  sourceId      string?                -- id of the source row
  scheduledAt   datetime?              -- null = immediate
  + envelope (createdAt, updatedAt)

  index (recipientId)
  index (scheduledAt)
  index (sourceType, sourceId)         -- find all notifs about one business object
  unique (dedupKey)                    -- partial: only where dedupKey is not null
  1 ─── * Delivery
```
