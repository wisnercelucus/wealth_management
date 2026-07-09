# Delivery

A single **physical send** of a [Notification](Notification.md) over one channel — the email to
`jean@…`, the SMS to `+509…`, the in-app push. One event produces **one Delivery per channel**; this
is where the channel, the rendered content, the lifecycle status and the retry/error state live. A
row is created `PENDING`, then advances as the channel and Celery workers act on it. Append-only:
retired via `status = CANCELLED`, never deleted (BRH 10-year audit).

## Essential fields

| Field          | Type              | Req | Description                                                                |
| -------------- | ----------------- | --- | ------------------------------------------------------------------------- |
| `id`           | uuid              | ●   | Surrogate PK (UUID4)                                                      |
| `notification` | FK→Notification   | ●   | The owning event (cascade delete), indexed                               |
| `channel`      | enum              | ●   | `Channel`: `IN_APP \| EMAIL \| SMS`                                      |
| `destination`  | string            | ○   | Resolved address for the channel (email / E.164 phone); `null` for IN_APP |
| `status`       | enum              | ●   | `Status` lifecycle (default `PENDING`), indexed                          |
| `title`        | string            | ○   | Rendered subject / heading **for this channel**                          |
| `body`         | text              | ○   | Rendered message body **for this channel**                               |
| `celeryTaskId` | string            | ○   | Celery task id, for revocation / tracing (EMAIL & SMS)                   |
| `error`        | text              | ○   | Last failure message (truncated)                                         |
| `retryCount`   | int               | ●   | Number of failed attempts (default `0`)                                  |
| `sentAt`       | datetime          | ○   | Stamp set on `SENT`                                                      |
| `deliveredAt`  | datetime          | ○   | Stamp set on `DELIVERED`                                                 |
| `readAt`       | datetime          | ○   | Stamp set on `READ` (in-app)                                            |
| + envelope     |                   | ●   | `createdAt` / `updatedAt` — see [README](README.md)                     |

## Notes & rules

- **One row per channel of one event.** `unique (notification, channel)` — an event never has two
  deliveries on the same channel.
- **Content is rendered per channel.** `title` / `body` come from the
  [NotificationTemplate](NotificationTemplate.md) matching `(notification.type, channel,
  notification.language)`, or from the explicit values the caller supplied. Rendered at creation and
  **frozen** — a later template edit never rewrites history.
- **Status lifecycle:** `PENDING → QUEUED → SENT → DELIVERED → READ`, plus terminal `FAILED` and
  `CANCELLED`. In-app jumps `PENDING → DELIVERED` (push); email/SMS go `PENDING → QUEUED → SENT →
  DELIVERED` via Celery. `READ` applies to IN_APP only.
- **Channel stamps are set by the channel, not the caller** — never write `sentAt` / `deliveredAt` /
  `readAt` directly; use the model's `mark_*` transitions.
- **`destination` is resolved at send** from the recipient's contact data in the consuming layer;
  it is stored on the row for audit and for the actual dispatch.
- **Suppression (non-critical only).** A delivery is created `CANCELLED` if its `channel` is not in
  the recipient's `activeChannels`, its event `type` is muted, or the recipient is inside quiet
  hours. `CRITICAL` notifications bypass all suppression.
- **PII / confidentiality.** `body` may contain client financial data; keep out of logs, restrict
  read access. Retained 10 years (BRH) — retire with `CANCELLED`, never delete.

## Clean model

```
Delivery
  id            uuid    PK
  notification  FK Notification         -- cascade delete, indexed
  channel       enum (IN_APP | EMAIL | SMS)
  destination   string?                 -- email / phone; null for IN_APP
  status        enum (PENDING | QUEUED | SENT | DELIVERED | READ | FAILED | CANCELLED)
  title         string?                 -- rendered for this channel
  body          text?                   -- rendered for this channel
  celeryTaskId  string?
  error         text?
  retryCount    int                     -- default 0
  sentAt        datetime?
  deliveredAt   datetime?
  readAt        datetime?
  + envelope (createdAt, updatedAt)

  unique (notification, channel)
  index (status)
  index (notification)
```
