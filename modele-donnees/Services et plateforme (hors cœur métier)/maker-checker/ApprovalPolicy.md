# ApprovalPolicy

The **configuration** — *how many approvals, from which people, in what order*, for one
`(targetType, operation)` pair. A policy is also the on/off switch of the whole module: **a flow is
under maker-checker if and only if an active policy exists for its `(targetType, operation)`.**
Gating a module is therefore adding a row, not shipping code.

A policy owns one or more [ApprovalStep](ApprovalStep.md) children. A policy with **zero steps** is
"logged but auto-approved" — the trail without the gate.

## Essential fields

| Field | Type | Req | Description |
|-------|------|-----|-------------|
| `id` | uuid | ● | Surrogate primary key. |
| `code` | string(50) | ● | Unique. |
| `labelFr` / `labelEn` | string | ● | |
| `targetType` | string(60) | ● | Which entity kind this policy governs. Matches [ChangeRequest](ChangeRequest.md)`.targetType`. |
| `operation` | enum | ● | Which operation it governs: `CREATE \| UPDATE \| DEACTIVATE \| POST`. |
| `mode` | enum | ● | `SEQUENTIAL` (steps satisfied in `orderIndex` order) \| `PARALLEL` (all steps open at once). |
| `thresholdField` | string(60) | ○ | A key of the payload to band on (e.g. `amount`). Required **iff** a bound is set. |
| `thresholdMin` | decimal(28,8) | ○ | Lower bound, **inclusive**. `NULL` = open below. |
| `thresholdMax` | decimal(28,8) | ○ | Upper bound, **exclusive**. `NULL` = open above. |
| `rejectionsToReject` | int | ● | How many `REJECT` decisions terminate the request. Default `1`. |
| `isActive` | bool | ● | Whether the policy is in force — the gate switch. Part of the envelope; called out because it is load-bearing here. |
| + envelope | | ● | See [README](README.md#shared-envelope). Ordinary mutable configuration. |

## Resolution — which policy governs a change

At submit, the module selects the **active** policy whose `(targetType, operation)` matches and whose
band contains the payload's `thresholdField` value. Unbanded policies (`thresholdField IS NULL`) are
the catch-all for that pair. If none matches, the change is **not gated**: the applier runs
immediately, and no request row is written. Once selected, the policy is pinned on the request.

Bands let a large transaction select a stricter policy — `(transaction, POST)` under 1 M HTG takes one
checker, above it takes two — without any conditional in the calling code.

## Notes & rules

- **Authoring guard: active bands may not overlap.** Two active policies on the same
  `(targetType, operation)` whose bands can both contain a value are rejected at save, as is a second
  unbanded policy on that pair. Ambiguous resolution is a configuration error, caught at authoring,
  never discovered at submit. The same discipline as [ChargePolicy](../../Cœur%20investissement/charges/ChargePolicy.md)'s
  double-tariffing guard.
- **A banded policy needs a numeric key.** `thresholdField` names a top-level key of the payload whose
  value is numeric. A submitted payload missing the key, or holding a non-numeric value, fails
  resolution and the submit is rejected — never silently treated as unbanded.
- **`SEQUENTIAL` opens step *n* only once every step before it is satisfied**; `PARALLEL` opens all
  steps at submit. Both are satisfied only when *every* step is.
- **Deactivating a policy does not touch in-flight requests.** They pinned it; they finish under it.
  `isActive = false` only means "govern no new changes". Never delete a policy that has requests.
- **Editing a policy's steps mid-flight is likewise invisible** to pinned requests, for the same
  reason. Retire and replace, rather than editing a policy that governs a live queue.
- **`operation` is an enum, and extending it costs nothing here.** `POST` was added for the ledger's
  gate. A new module-specific verb is a new enum value; the module gains no knowledge of what it means.
- **Zero steps = auto-approve.** The request is written, `SUBMITTED` and `APPROVED` in the same breath,
  the applier runs, `APPLIED`. Useful to build the audit trail for a flow before deciding who checks it.

## Deliberately absent

- **No `targetType` FK, no target table.** As everywhere in this module.
- **No `requiresChecker` flag on the governed entity.** The presence of the policy *is* the flag; a
  duplicate boolean on `TransactionType` or `Portfolio` would be a second source of truth that drifts.
- **No per-policy enable/disable schedule.** Gate configuration is not tariff configuration: there is
  no `effectiveFrom` / `effectiveTo`. A gate is on or off, now.

## Clean model

```
ApprovalPolicy
  id                  uuid    PK
  code                string(50)  unique
  labelFr             string
  labelEn             string
  targetType          string(60)
  operation           enum (CREATE | UPDATE | DEACTIVATE | POST)
  mode                enum (SEQUENTIAL | PARALLEL)
  thresholdField      string(60)?      -- required iff a bound is set
  thresholdMin        decimal(28,8)?   -- inclusive
  thresholdMax        decimal(28,8)?   -- exclusive
  rejectionsToReject  int  default 1
  + envelope (isActive, externalId, externalRef, createdAt, createdBy, modifiedAt, modifiedBy)

  index (targetType, operation) where isActive
  check (thresholdMin is null and thresholdMax is null) or (thresholdField is not null)
  check (thresholdMin is null or thresholdMax is null or thresholdMin < thresholdMax)
  check (rejectionsToReject >= 1)
  -- authoring guard (not expressible as a constraint): no two active policies on the same
  -- (targetType, operation) may have overlapping bands, and at most one may be unbanded.

  1 ─── * ApprovalStep
  1 ─── * ChangeRequest
```
