# ApprovalStepEligibility

**Which approvers** may satisfy a [step](ApprovalStep.md). One row per eligible principal, where a
principal is a role, a group, or a named user. A step's eligible set is the union of its rows,
resolved through the identity lookup at decision time.

Listing a role and two users on the same step is ordinary: *"any Conseiller principal, or Marie, or
Jean."*

## Essential fields

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `stepId` | FK→ApprovalStep | ● | The step whose set this row extends. Cascade. |
| `principalType` | enum | ● | `ROLE \| GROUP \| USER`. |
| `principalId` | string(60) | ● | The role code, group code, or user id — a **soft reference**, no FK. |
| + envelope | | ● | See [README](README.md#shared-envelope). Ordinary mutable configuration. |

## Notes & rules

- **`principalType` + `principalId` is a polymorphic soft reference**, the same shape as
  [ChangeRequest](ChangeRequest.md)`.targetType` / `targetId`. The module holds no FK into the security
  layer's role and group tables: it asks identity *"does user U hold principal P?"* and believes the
  answer. That is contract 2 of [the seam](README.md#the-seam).
- **The union is a union, never an intersection.** A user eligible through *any* row of the step may
  decide on it. A step that must require two different qualifications is two steps.
- **Eligibility is resolved at decision time, not at submit.** A checker promoted into the compliance
  role this morning can clear this afternoon's queue; one who left the role can no longer decide, even
  on a request that was submitted while they held it. Approvals already recorded stand — a decision is
  frozen truth about the moment it was made.
- **Deactivating an eligibility row does not retract decisions** it once authorized. It narrows the
  set for every decision not yet taken, including on in-flight requests: unlike steps, the eligible
  set is deliberately *not* pinned, because it tracks who is currently entitled to act.
- **The maker is filtered out of the resolved set**, whatever the rows say. See
  [ApprovalDecision](ApprovalDecision.md).
- **An empty set makes the step unsatisfiable.** A step with no active eligibility rows is an authoring
  error; the module rejects saving a policy in that shape.

## Deliberately absent

- **No `isRequired` or per-principal count.** "Two from this role, one from that one" is two steps, not
  two weighted rows. Weighting a set is how an approval matrix becomes unreadable.
- **No FK to Role / Group / User.** The foundational tier does not import the security layer's tables;
  it consults its lookup.
- **No denial rows.** There is no *ineligible* principal — a set is defined by what it contains.

## Clean model

```
ApprovalStepEligibility
  id             uuid    PK
  stepId         FK ApprovalStep     -- cascade
  principalType  enum (ROLE | GROUP | USER)
  principalId    string(60)          -- soft ref → security layer, no FK
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)

  unique (stepId, principalType, principalId) where isActive
  index  (principalType, principalId)   -- "which steps can this role clear?"

  * ─── 1 ApprovalStep
```
