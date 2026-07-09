# NotificationTemplate

The FR/EN message gabarits, resolved by **`type × channel × language`**. The service looks one up
at send time, renders it against the caller's context, and copies the result onto each
[Delivery](Delivery.md) row — so templates are an *input* to message creation, never linked to the
produced rows.

## Essential fields

| Field            | Type        | Req | Description                                                            |
| ---------------- | ----------- | --- | --------------------------------------------------------------------- |
| `id`             | bigint      | ●   | Surrogate PK                                                          |
| `type`           | string (free)| ●  | Free type, must match the `type` the caller passes to `send()`       |
| `channel`        | enum        | ●   | `Channel`: `IN_APP \| EMAIL \| SMS`                                  |
| `language`       | enum        | ●   | `Language`: `fr \| en` (default `fr`)                                |
| `titleTemplate`  | string      | ○   | Django-template string for the title (rendered with `context`)      |
| `bodyTemplate`   | text        | ○   | Django-template string for the body                                  |
| `isActive`       | bool        | ●   | Only active templates are resolved (default `true`)                  |
| + envelope       |             | ●   | `createdAt` / `updatedAt` — see [README](README.md)                 |

## Notes & rules

- **Unique key `(type, channel, language)`** — at most one active gabarit per combination; this is
  exactly the lookup the service performs.
- **Django template syntax.** `titleTemplate` / `bodyTemplate` are rendered with the context dict
  passed to `send_from_template(...)` (e.g. `{{ ticker }}`, `{{ change }}`). Render is sandboxed to
  the supplied context — no DB/model access.
- **The render context IS the notification's `metadata`.** The keys the caller puts in
  [Notification](Notification.md)`.metadata` (`ticker`, `montant`…) are exactly the template
  variables — so `metadata` serves a double role: audit context and render variables.
- **No FK to Delivery.** The rendered output is copied onto the delivery at creation; editing a
  template never rewrites existing messages.
- **Fallback.** If no active template matches `(type, channel, language)`, the service falls back to
  the explicit `title` / `body` supplied by the calling app (which always supplies both).
- **i18n.** Provide one row per language; the recipient's `UserPreferences.language` (or the
  caller's override) selects which is used.

## Clean model

```
NotificationTemplate
  id            bigint  PK
  type          string  -- free; must match the caller's send() type
  channel       enum (IN_APP | EMAIL | SMS)
  language      enum (fr | en)
  titleTemplate string?               -- Django template string
  bodyTemplate  text?                 -- Django template string
  isActive      bool                  -- default true
  + envelope (createdAt, updatedAt)

  unique (type, channel, language)
```
