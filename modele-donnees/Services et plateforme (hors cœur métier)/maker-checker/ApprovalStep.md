# ApprovalStep

**One requirement within a [policy](ApprovalPolicy.md)** — *how many approvers*. A step is satisfied
when `requiredCount` distinct eligible principals have each recorded an `APPROVE`
[decision](ApprovalDecision.md) against it. A policy is satisfied when all of its steps are.

*Which* approvers is the step's [eligibility](ApprovalStepEligibility.md) set, a child table, so one
step can name a role, a group, and a handful of specific users at once.

## Essential fields

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `policyId` | FK→ApprovalPolicy | ● | The policy this step belongs to. Cascade: a step never outlives its policy. |
| `orderIndex` | int | ● | Position in the chain. Read only when the policy is `SEQUENTIAL`; still required (and unique per policy) so a policy can be switched to `SEQUENTIAL` without re-authoring. |
| `label` | string | ○ | What this step means to a human — "Validation Conformité", "Contrôle Direction". Shown on the checker's queue. |
| `requiredCount` | int | ● | How many approvals this step needs from its eligible set. Default `1`. |
| `distinctApprover` | bool | ● | If true, whoever satisfies this step may not satisfy another step of the same request — true N-eyes. Default `false`. |
| + envelope | | ● | See [README](README.md#shared-envelope). Ordinary mutable configuration. |

## Notes & rules

- **`requiredCount` counts people, not decisions.** An approver counts at most once toward a step
  (`UNIQUE (changeRequestId, stepId, approverId)` on the decision), so two clicks are one approval.
- **The eligible set must be large enough to satisfy the step.** A step requiring three approvals whose
  eligibility resolves to two users can never be satisfied. The set is resolved through the identity
  lookup at *decision* time, not at authoring, so this is a runtime deadlock rather than an authoring
  error — the pending queue exposes it (a request whose remaining steps have no eligible approver left
  is surfaced, never silently stuck).
- **`distinctApprover` is a whole-request constraint, not a step-local one.** When set, a principal who
  already approved *any* step of that request is ineligible here. Two steps both marked
  `distinctApprover` therefore need two different people even if the same role is eligible for both.
  This is what "four eyes" actually means when the same director sits in two eligibility sets.
- **The maker is never eligible.** Separation of duties is enforced on the decision, above whatever the
  eligibility rows say — see [ApprovalDecision](ApprovalDecision.md).
- **`SEQUENTIAL` reads `orderIndex`; `PARALLEL` ignores it.** Under `SEQUENTIAL`, step *n* accepts
  decisions only once every step with a lower `orderIndex` is satisfied; a decision arriving early is
  rejected, not queued.
- **Steps are configuration, and configuration is pinned.** Adding a step to a policy that governs
  in-flight requests does not lengthen them: they were pinned to the policy at submit and are evaluated
  against the steps as they stand — which is why a policy under a live queue is retired and replaced,
  not edited. See [ApprovalPolicy](ApprovalPolicy.md).
- **Zero steps is a policy-level statement**, not a step with `requiredCount = 0`. `requiredCount >= 1`
  always.

## Deliberately absent

- **No `state`.** A step's satisfaction is computed from decisions. There is no per-request step-state
  row anywhere in this module, by design.
- **No approver columns.** Eligibility is a set, and a set is a child table — a step never names one
  person in a column.
- **No `isOptional`.** An optional requirement is not a requirement; author the policy without it.

## Clean model

```
ApprovalStep
  id                uuid    PK
  policyId          FK ApprovalPolicy   -- cascade
  orderIndex        int
  label             string?
  requiredCount     int   default 1
  distinctApprover  bool  default false
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)

  unique (policyId, orderIndex)
  check  (requiredCount >= 1)

  * ─── 1 ApprovalPolicy
  1 ─── * ApprovalStepEligibility
  1 ─── * ApprovalDecision
```
