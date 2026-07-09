# Notifications cluster Model Reference

A **generic delivery engine** the rest of the platform calls directly via
`NotificationService.send()`. It knows **no business concept** — the calling app (e.g. ProFin)
decides *what* to notify and supplies a free-form `type` plus its context. Each call creates **one
`Notification`** (the logical event) and **one `Delivery` per channel** (the physical send), so a
margin call by email *and* SMS is one event with two deliveries. Identity stays in the **Party**
cluster: notifications only hold a soft `recipientId` (the external user id), never a FK into auth.
_Produce the event once, route and track it per channel._

## Models

| Model                                             | Purpose                                                          |
| ------------------------------------------------- | --------------------------------------------------------------- |
| [Notification](Notification.md)                   | The logical event record (one row per event)                    |
| [Delivery](Delivery.md)                           | A physical send over one channel (one row per channel, 1:many)  |
| [NotificationTemplate](NotificationTemplate.md)   | FR/EN message gabarits, resolved by `type × channel × language` |
| [UserPreferences](UserPreferences.md)             | Per-user channels, language, timezone, muted types (1:1 by user)|
| [QuietHours](QuietHours.md)                        | Silence windows for a user's preferences (1:many)               |

## Relationships

```
Notification 1───* Delivery                            (FK, the only internal join)
UserPreferences 1───* QuietHours

Notification         *───1  recipientId → Party/User   (soft reference, no FK)
NotificationTemplate        resolved by (type × channel × language)  -- no FK, an input to Delivery

(no Django signals: the calling app invokes NotificationService.send() directly)
```

> The **only** FK inside the module is `Delivery → Notification` (an event owns its channel sends).
> `NotificationTemplate` and `UserPreferences` stay **independent** — never joined by FK. The service
> resolves a template at send time and copies the rendered `title` / `body` onto each **Delivery**
> row, so the message stays immutable even if the template later changes. Preferences are looked up
> by `recipientId` / `userId`, again without a FK.

## Type & enums (local to this module)

`type` is a **free `CharField`** (no `choices`) on both `Notification` and `NotificationTemplate` —
the engine never constrains it; the calling app passes whatever string it wants. For convenience,
[`enums.py`](../../notifications/enums.py) exposes **suggested, non-binding** generic categories via
`NotificationCategory`: `ALERT | OPERATION | CONTENT | SYSTEM`. (Business-specific values such as
`ORDER_EXECUTED` or `MARGIN_CALL` live in the calling app, not here.)

The remaining enums are Django `TextChoices` (DB value = plain string):

- `Channel` — `IN_APP | EMAIL | SMS` (on **Delivery**)
- `Priority` — `LOW | NORMAL | HIGH | CRITICAL` (on **Notification**; CRITICAL bypasses muting / quiet hours)
- `Status` — `PENDING → QUEUED → SENT → DELIVERED → READ` (+ `FAILED`, `CANCELLED`) (on **Delivery**)
- `Language` — `fr | en` (on **Notification**)

## Reference FKs (→ référentiel / other clusters)

- `recipientId` → **Party** cluster (soft reference — string id, no DB FK, keeps the module decoupled).
- `language` uses the shared `Language` enum aligned with the **référentiel** layer.
- `issuerCountryId` and other référentiel FKs are **not** used here — notifications carry no
  reference-data foreign keys; contextual data travels in the free-form `metadata` JSON.

## Shared envelope — on every table

Unlike the Party cluster, the notifications module does **not** use `isActive` soft-delete or
`externalId` — notifications are an immutable, append-only audit log (BRH 10-year retention) and are
retired via the delivery `status` (`CANCELLED`), never deleted. The common envelope is the audit
timestamps:

```
id          uuid (Notification, Delivery) / bigint (others)   PK
createdAt   datetime    auto, set on insert
updatedAt   datetime    auto, set on every save
```

`Delivery` additionally carries lifecycle stamps (`sentAt`, `deliveredAt`, `readAt`) — see its page.
`QuietHours` has no timestamps (it is owned by, and cascades with, `UserPreferences`).
