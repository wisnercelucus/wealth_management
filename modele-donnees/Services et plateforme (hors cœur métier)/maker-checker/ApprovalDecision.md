# ApprovalDecision

**The record of each act** — one row per checker, per step, per decision. Approvals accumulate against
the [steps](ApprovalStep.md) of the pinned [policy](ApprovalPolicy.md) until every step is satisfied
and the [request](ChangeRequest.md) is approved; a rejection stops it.

These rows are the module's only state. Everything the module reports — *is this step satisfied, is
this request approved, who signed off on it* — is computed from them. Audit-grade, append-only,
frozen: a decision is never edited, withdrawn, or deleted (BRH 10-year retention).

## Essential fields

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `changeRequestId` | FK→ChangeRequest | ● | The request being decided. |
| `stepId` | FK→ApprovalStep | ● | Which requirement this decision answers. Must belong to the request's pinned policy. |
| `approverId` | FK→User | ● | Who decided. Never the request's `makerId`. |
| `decision` | enum | ● | `APPROVE \| REJECT`. |
| `comment` | string | ○ | Optional note. Expected on a `REJECT` — the maker reads it before re-proposing. |
| `decidedAt` | datetime | ● | When. |
| + envelope | | ● | See [README](README.md#shared-envelope). Permanent record: `isActive` fixed `true`, `modifiedAt` / `modifiedBy` stay null. |

## The rules enforced at decision time

A decision is admitted only if all of these hold. Each is checked against the request's **pinned**
policy, not the current configuration.

- **Separation of duties.** `approverId != changeRequest.makerId`. A maker never approves their own
  request, whatever the eligibility rows say. This one is absolute and has no configuration.
- **State.** The request is `SUBMITTED`. A `DRAFT` has no checkers; a terminal request takes no more
  decisions.
- **Eligibility.** The approver holds at least one principal in the step's
  [eligibility](ApprovalStepEligibility.md) set, resolved now through the identity lookup.
- **Once per step.** `UNIQUE (changeRequestId, stepId, approverId)` — a second click is not a second
  approval.
- **N-eyes.** If the step's `distinctApprover` is set, the approver has no decision on any other step
  of this request.
- **Order.** If the policy's `mode` is `SEQUENTIAL`, every step with a lower `orderIndex` is already
  satisfied. A decision arriving out of order is refused, not held.

## Notes & rules

- **Satisfaction is computed, never stored.** A step is satisfied when its `APPROVE` rows count
  `requiredCount` distinct approvers; the request is approved when every step of the pinned policy is.
  There is no `currentStep`, no per-request step-state table, no counter — those would be banned
  derivables, and the ones that drift first.
- **Rejection is terminal by default.** The `rejectionsToReject`-th `REJECT` row (default: the first)
  moves the request to `REJECTED` and `resolvedAt` is stamped. A rejected request is never revived —
  the maker raises a new one, and the rejection stays in the trail attached to the old one.
- **A rejection still names a step.** It is recorded against the step the approver was eligible for, so
  the trail answers *at which control the change died*, not merely *that it did*.
- **Withdrawal is not a decision.** A maker who changes their mind cancels the request
  (`state = CANCELLED`); an approver who changes theirs cannot — the row stands, and the request is
  cancelled and re-proposed. Freezing the act is the point of the record.
- **The final approval and the apply are one database transaction.** The decision that satisfies the
  last step, the flip to `APPLIED`, and the applier's domain writes commit together or not at all. An
  applier failure rolls the decision back with it: the checker sees an error and clicks again, rather
  than finding an approved change that never happened.
- **Approvals recorded under a since-revoked eligibility stand.** Eligibility is judged when the
  decision is made; the row is truth about that moment. See
  [ApprovalStepEligibility](ApprovalStepEligibility.md).
- **Entering `APPROVED` and `REJECTED` emits a lifecycle event**; notifications and audit subscribe.
  This module writes no notification rows and sends no mail.

## Deliberately absent

- **No `state` or `isCurrent`.** A decision is an event, not an entity with a life.
- **No reversal, no counter-decision.** Unlike the ledger, which reverses a posted transaction with an
  offsetting one, an approval has nothing to offset: the request it fed either applied or did not.
- **No `stepOrderIndex` snapshot.** The step is pinned through the request's policy; read it there.

## Clean model

```
ApprovalDecision
  id               uuid    PK
  changeRequestId  FK ChangeRequest
  stepId           FK ApprovalStep       -- must belong to the request's pinned policy
  approverId       FK User
  decision         enum (APPROVE | REJECT)
  comment          string?
  decidedAt        datetime
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)

  unique (changeRequestId, stepId, approverId)   -- once per step
  index  (changeRequestId)
  index  (approverId, decidedAt)                 -- a checker's own trail
  -- gate rules (not expressible as constraints): approverId != request.makerId;
  -- request.state = SUBMITTED; approver eligible for the step; distinctApprover honoured;
  -- SEQUENTIAL order respected.

  * ─── 1 ChangeRequest
  * ─── 1 ApprovalStep
```
