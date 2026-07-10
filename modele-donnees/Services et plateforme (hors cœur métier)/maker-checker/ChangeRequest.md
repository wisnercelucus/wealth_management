# ChangeRequest

The **unit of approval** — one row per proposed change to an opaque target. A maker submits *"I want
to `{operation}` a `{targetType}` — here is the proposed data"*, and this row parks that intent until
the governing [policy](ApprovalPolicy.md) is satisfied. It carries the change; it never *is* the
change. **A change request has no effect on the system until it reaches `APPLIED`**: the pending
instrument is not an instrument, the pending transaction moves no position.

Audit-grade, append-only (BRH 10-year retention): a resolved request is never edited or deleted; the
maker raises a new one.

## Essential fields

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `targetType` | string(60) | ● | What kind of entity the change is about (`portfolio`, `instrument`, `party`, `transaction`…). A plain identifier, **never a FK** — the module does not know the target's table. |
| `targetId` | string | ○ | The existing entity being changed. `NULL` on `CREATE` until the applier returns the new id — see below. Not a FK. |
| `operation` | enum | ● | `CREATE \| UPDATE \| DEACTIVATE \| POST`. |
| `payload` | json | ● | The proposed change — **opaque**. The owning module serializes it on submit and reads it on apply. |
| `policyId` | FK→ApprovalPolicy | ● | The policy resolved at submit and **pinned** for the life of the request. |
| `state` | enum | ● | `DRAFT \| SUBMITTED \| APPROVED \| APPLIED \| REJECTED \| CANCELLED`. |
| `makerId` | FK→User | ● | Who proposed the change. Never an approver on this request. |
| `submittedAt` | datetime | ○ | When it entered `SUBMITTED`. |
| `resolvedAt` | datetime | ○ | When it reached `APPLIED` / `REJECTED` / `CANCELLED`. |
| `narrative` | string | ○ | The maker's free-text justification, shown to checkers beside the rendered payload. |
| + envelope | | ● | See [README](README.md#shared-envelope). Permanent record: `isActive` fixed `true`, `modifiedAt` / `modifiedBy` stay null. |

## Notes & rules

- **`targetType` + `targetId` is a polymorphic soft reference.** No FK, no generic foreign key. It is
  what lets one mechanism gate every entity in the platform; the applier registry resolves the type
  at apply time. The same pattern as `Notification.sourceType` / `sourceId`.
- **`targetId` is stamped on `CREATE` at apply time.** A create has no target when it is proposed. The
  applier's `apply(payload) → id` returns the row it wrote, and that id lands here in the same
  database transaction as the flip to `APPLIED`. So a `CREATE` request is the permanent audit link
  between the proposal and the entity it produced.
- **`policyId` is pinned at submit, not resolved at read.** Charge policies are reconstructed from
  validity dates because nothing accumulates against them; approvals do. A request that collected two
  of three approvals must keep being judged against the three-step policy even if an admin edits the
  configuration mid-flight. Pinning is what makes an in-flight request deterministic.
- **`payload` is opaque and is never validated here.** The applier's `validate(payload)` runs at
  submit (a malformed change never reaches a checker) and again at apply (the world moved). The
  applier's `render(payload)` is what a checker actually reads — this module cannot describe a change
  it does not understand.
- **One in-flight request per target and operation.** A partial `UNIQUE (targetType, targetId,
  operation) where state = SUBMITTED` stops two makers from queueing conflicting edits to the same
  portfolio. `CREATE` requests carry a null `targetId` and are exempt — concurrent creates are
  legitimate.
- **`DRAFT` is optional.** A maker may save a proposal before submitting it; nothing is resolved, no
  policy is pinned, no checker sees it. Most flows submit directly.
- **The terminal states are terminal.** `APPLIED`, `REJECTED` and `CANCELLED` are never left. A
  rejected change is re-proposed as a new request, which keeps the rejection in the trail.
- **`CANCELLED` belongs to the maker only.** Withdrawing is not a decision and writes no
  [ApprovalDecision](ApprovalDecision.md) row.
- **Approval and application are one database transaction.** On the final approval the applier writes
  the domain rows and the request flips `APPROVED → APPLIED` atomically. `APPROVED` is therefore a
  state one rarely observes at rest; it exists so that an applier failure leaves an approved-but-
  unapplied request to retry rather than a silently lost decision.
- **Lifecycle events.** Entering `SUBMITTED`, `APPROVED` and `REJECTED` emits an event that the
  notifications and audit modules subscribe to. This module sends nothing itself.
- **PII / confidentiality.** `payload` carries whatever the owning module put there — client data,
  amounts, document numbers. Keep out of logs; restrict read access to the request's eligible
  approvers and its maker. Retained 10 years (BRH).

## Deliberately absent

- **No per-request step state.** Whether a step, and the whole request, is satisfied is **computed**
  from its [decisions](ApprovalDecision.md) against the pinned policy's [steps](ApprovalStep.md). A
  stored `currentStep` would be a banned derivable, and the one that goes stale first.
- **No `approvedAt`.** It is `resolvedAt` on the `APPLIED` row, or the last decision's `decidedAt`.
- **No rendered summary.** `render(payload)` is a call, not a column: the owning module's language
  changes, the payload does not.
- **No FK to the target.** Deliberate — it is the whole premise of the module.

## Clean model

```
ChangeRequest
  id           uuid    PK
  targetType   string(60)          -- soft ref, no FK: portfolio | instrument | party | transaction …
  targetId     string?             -- null on CREATE until APPLIED stamps the applier's returned id
  operation    enum (CREATE | UPDATE | DEACTIVATE | POST)
  payload      json                -- opaque to this module
  policyId     FK ApprovalPolicy   -- pinned at submit
  state        enum (DRAFT | SUBMITTED | APPROVED | APPLIED | REJECTED | CANCELLED)
  makerId      FK User
  submittedAt  datetime?
  resolvedAt   datetime?
  narrative    string?
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)

  index (targetType, targetId)                    -- the audit trail of one business object
  index (state, submittedAt)                      -- a checker's pending queue
  index (makerId)
  unique (targetType, targetId, operation) where state = SUBMITTED   -- partial; CREATE exempt (targetId null)
  check  (state = 'DRAFT') or (submittedAt is not null)
  check  (state not in ('APPLIED','REJECTED','CANCELLED')) or (resolvedAt is not null)
  check  (operation != 'CREATE') or (state != 'APPLIED') or (targetId is not null)

  * ─── 1 ApprovalPolicy
  1 ─── * ApprovalDecision
```
