# The Maker-Checker Contract (`approbation`)

This document defines the **contract** between a caller — any domain module that wants a change gated
— and the maker-checker module. It is the agreement both sides sign so that one generic mechanism can
wrap creating an instrument, editing a portfolio, deactivating a client or posting a transaction
without `approbation` ever importing a single domain entity.

Read [requirements.md](requirements.md) and [README.md](README.md) first; this file zooms in on one
sentence of the README — *"The module integrates through exactly three contracts"* — and makes it
precise enough to build against.

---

## 0. First, what a "contract" actually is

It is worth clearing up a common half-truth, because it is the exact half-truth this document exists to
fix.

If you have built approval flows in Django before, you probably reached for the `ContentType` framework:
you passed the flow an `app_label`, a `model` (together, the `ContentType`) and an `object_id`, and hung
the approval off that generic pointer. That instinct is **correct but incomplete**.

`(app_label, model, object_id)` is the **addressing** half of a contract — the *what* and *where*. It
tells the approval flow *which object* a request concerns. In this module that maps almost one-to-one:

| Django `GenericForeignKey`        | maker-checker            |
| --------------------------------- | ------------------------ |
| `content_type` (`app_label.model`) | `targetType` (a string)  |
| `object_id`                       | `targetId` (a string)    |

But addressing is not the whole contract, and here is the tell: in Django you never had to ask *"who
writes the object once it's approved?"* — you just called `obj.save()` in the same process, against the
same ORM, because your approval app was allowed to import your models. **This module is not.** It imports
nothing domain-specific — that is the entire reason it can gate everything. So it *cannot* call `.save()`
for you.

That means the caller has to hand the module a second thing the Django pattern never made you think
about: **a way to actually perform the write**. That is the *behavioral* half of the contract — the
**applier**. A contract, properly stated, is therefore three things, not one:

1. **The call envelope** — the opaque proposal the caller *sends in* (this is the `ContentType` part you
   already know).
2. **The applier interface** — the callbacks the caller *implements and registers*, so the module can
   validate, render and eventually write the change without knowing what it is.
3. **The promises** — the guarantees each side makes about state, atomicity and events, so neither side
   has to trust the other's internals.

Your Django mental model had #1. The teammate asking for "a contract" is asking for **#2 and #3** —
which is where all the real decoupling lives.

> **A contract is not the data you pass. A contract is the full bilateral agreement: what the caller
> provides, what the caller promises to implement, and what the module promises back.**

---

## 1. The shape of the contract at a glance

```
CALLER (owning module)                         MAKER-CHECKER (approbation)
──────────────────────                         ───────────────────────────
  builds the envelope   ──── submit_change ───▶  parks a ChangeRequest
  (targetType,                                    resolves the policy
   targetId, operation,                           collects ApprovalDecisions
   payload)                                       enforces the rules
                                                        │
  registers an applier  ◀─── validate/render ───        │  (checkers read render())
  keyed by targetType         at submit & display        │
                                                        ▼
  applier.apply(payload) ◀──── apply ───────────  on final approval, in ONE txn:
  writes the domain rows,                          apply() + flip to APPLIED
  returns the new id     ──── id ──────────────▶  stamp targetId, emit events
```

Two directions cross the seam. **Data flows in** as an opaque envelope. **Behaviour flows in** as a
registered applier the module calls back. Nothing domain-specific ever flows the other way — the module
returns only ids, states and events.

---

## 2. Part A — The call envelope (the data contract)

This is what the caller passes to `submit_change(...)`. It is the [ChangeRequest](ChangeRequest.md)
seen from the caller's side. Every gated write path builds one of these and hands it over
**unconditionally** — whether or not a gate is actually configured is the module's decision, not the
caller's (see §6).

| Field        | Type   | Req | Who fills it | Meaning                                                                                       |
| ------------ | ------ | --- | ------------ | --------------------------------------------------------------------------------------------- |
| `targetType` | string | ●   | caller       | The kind of entity — `instrument`, `portfolio`, `client`, `transaction`. A stable identifier. |
| `targetId`   | string | ○   | caller       | The id of the existing entity for `UPDATE` / `DEACTIVATE`. **`NULL` for `CREATE`.**           |
| `operation`  | enum   | ●   | caller       | `CREATE \| UPDATE \| DEACTIVATE \| POST`. Combined with `targetType` to resolve the policy.    |
| `payload`    | json   | ●   | caller       | The proposed change — **opaque to the module**. See the opacity and self-containment rules.    |
| `makerId`    | FK→User | ●  | caller       | Who is proposing. The module enforces maker ≠ approver against this.                            |
| `thresholdField` value | (in payload) | ○ | caller | If a policy bands on a payload key (e.g. `amount`), the caller must put that key in the payload. |

Three properties of the envelope are load-bearing and easy to get wrong:

### 2.1 `targetType` is a *reserved string*, not a table

It is not a foreign key. The module never joins on it and never validates it against a registry of
entities. It is the **key under which the caller registered its applier** (§3) *and* the key on which
[ApprovalPolicy](ApprovalPolicy.md) is configured. So it must be:

- **Stable** — never rename `instrument` to `financial_instrument` later; existing requests and policies
  are pinned to the old string.
- **Namespaced by the owning module** — one module, one prefix; avoid two modules both claiming
  `account`.
- **Agreed once** — it is the public name of your entity in the approval system. Treat it like a wire
  format, not an internal class name. (This is the one respect in which it is *safer* than Django's
  `ContentType`: it is a plain agreed string, not a row in `django_content_type` that a migration can
  renumber.)

### 2.2 The `payload` is opaque — and that is a rule the module actually enforces

The module never opens the payload. It does not read it, band on it (except through the caller-declared
`thresholdField` key, treated as an uninterpreted number), diff it, or log its interior. The **only**
things that ever understand the payload are the caller's own `validate`, `render` and `apply`.

Consequence for the caller: **the payload must be self-contained.** It is captured at submit and applied
possibly hours or days later, after N approvals. At apply time the module hands your `apply()` *exactly*
the bytes you submitted — nothing more. If the write needs a value, that value must be inside the
payload. Do not stash half the change in the payload and half in some request-scoped context that will
be gone by apply time.

### 2.3 The payload should carry a version and an optimistic-lock token

Because the payload is frozen at submit but written at apply, two clocks can drift between them:

- **Your own schema can move.** If you deploy a payload-format change while old pending requests exist,
  a naïve `apply()` will misread the old shape. Put a `payloadVersion` inside the payload and branch on
  it in `apply()`. (This is the same discipline the notifications module uses when it freezes rendered
  title/body so template edits never rewrite history.)
- **The target can move.** For `UPDATE` / `DEACTIVATE`, someone may change the underlying entity between
  submit and apply. If you blindly write the approved payload, you silently clobber the newer state. Put
  the entity's version/etag you read at submit into the payload as an **expected version**, and have
  `apply()` reject if the live entity has moved past it. This turns a lost-update into a clean failure
  the maker can re-raise — structurally the same instinct that kills "the Axia 12-day desync bug" in the
  position module.

Neither of these is the module's job — it cannot do them, it can't read the payload. They are **your
side of the contract**, and this document names them so they are not forgotten.

---

## 3. Part B — The applier (the behavioral contract)

This is the half the Django `ContentType` pattern hides. Each owning module implements an **applier** and
**registers** it under its `targetType`. The module looks it up by `targetType` at the two moments it
needs the caller's help, having imported nothing.

An applier is three functions:

| Method               | Called when                              | Returns | Contract                                                                                          |
| -------------------- | ---------------------------------------- | ------- | ------------------------------------------------------------------------------------------------- |
| `validate(payload)`  | at **submit**, and again at **apply**    | ok/errors | Is the proposed change well-formed and still applicable? Pure check, **no writes.**             |
| `render(payload)`    | whenever a checker views the request     | display | Present the change in the domain's own language so a checker knows what they approve.            |
| `apply(payload) → id` | once, on **final approval**, inside the module's transaction | the affected entity id | Perform the actual domain write. Return the id — a `CREATE` stamps it into `targetId`. |

### 3.1 Why `validate` runs twice

Once at **submit**, to fail fast — a malformed change should never enter the queue and waste a checker's
time. Once again at **apply**, because the world may have changed under an `UPDATE` while it sat awaiting
approval (see the optimistic-lock token in §2.3). The second call is the module protecting the caller
from applying a now-invalid change. Make `validate` cheap and side-effect-free so calling it twice costs
nothing.

### 3.2 Why `apply` returns the id

For a `CREATE`, the entity does not exist until `apply` writes it, so `targetId` is `NULL` on the way in.
`apply` returns the freshly-minted id and the module stamps it into `ChangeRequest.targetId`, so the
audit trail, the events, and any later reference point at a real row. For `UPDATE` / `DEACTIVATE`, return
the same id you were given — it confirms the write landed where expected.

### 3.3 The registry is the dependency inversion

```
domain module  ──registers──▶  approbation's applier registry   (keyed by targetType)
approbation    ──looks up───▶  applier                          (imports nothing)
```

The arrow points **from** the domain module **to** `approbation`, never back. `approbation` depends on no
domain module; domain modules depend on `approbation`. That one-way arrow is the whole reason a single
module can gate the entire platform. It is the same inversion the report calls out as the system's
recurring strength: *the dependency arrow points away from the ledger.*

---

## 3bis. How a module actually implements the applier

The theory above is short; here is what it looks like in practice, because this is where it clicks. Three
facts do all the work:

**1. One applier per module, registered once at startup — not one per object, not one per operation.**

Each owning module writes a single applier and registers it under its `targetType` when it boots:

```
# instrument module, at startup
approbation.register_applier("instrument", InstrumentApplier())
# portfolio module, at startup
approbation.register_applier("portfolio", PortfolioApplier())
# client module, at startup
approbation.register_applier("client", ClientApplier())
```

`approbation` now holds a tiny lookup table and nothing else about your domain:

| targetType | applier |
| ---------- | ----------------- |
| instrument | InstrumentApplier |
| portfolio  | PortfolioApplier  |
| client     | ClientApplier     |

When a request for `targetType = "instrument"` reaches final approval, `approbation` looks up that row
and calls `InstrumentApplier.apply(payload)`. It never imported `Instrument` — it just calls whatever
object the instrument module handed it. **That lookup is the entire trick.**

**2. The one applier handles every operation by branching on `operation` inside.** `CREATE`, `UPDATE`
and `DEACTIVATE` for instruments all go through the same `InstrumentApplier`:

```
class InstrumentApplier:

    def validate(self, payload):        # no writes — called at submit AND again at apply
        if payload["operation"] == "CREATE":
            assert Instrument.code_is_free(payload["code"])
        else:                            # UPDATE / DEACTIVATE
            assert Instrument.exists(payload["targetId"])

    def render(self, payload):          # human summary for the checker
        return f'Create bond {payload["code"]}'

    def apply(self, payload):           # THE write — approbation calls this on final approval
        op = payload["operation"]
        if op == "CREATE":
            row = Instrument.create(code=payload["code"], type=payload["type"], ...)
            return row.id                # stamped back into targetId
        elif op == "UPDATE":
            Instrument.update(payload["targetId"], name=payload["name"])
            return payload["targetId"]
        elif op == "DEACTIVATE":
            Instrument.deactivate(payload["targetId"])
            return payload["targetId"]
```

Every other module writes the **same three-method shape** with its own domain logic inside —
`PortfolioApplier.apply` writes the portfolio table, `ClientApplier.apply` writes the client table.
Implementing the contract *is* exactly this: write three methods, register them. Everything about *what
an instrument is* stays inside the instrument module; `approbation` stays ignorant.

**3. `apply` is only ever reached on the fully-approved path.** Every other outcome leaves your domain
table untouched, because the proposal only ever lived as JSON in `change_request`:

| Outcome                                     | Is `apply` called? | Does the domain table change?                          |
| ------------------------------------------- | ------------------ | ------------------------------------------------------ |
| Approved — all steps satisfied              | **Yes**, once, in one transaction | Yes — the real row appears                |
| Rejected                                    | **No**             | No — never touched                                     |
| `apply` throws (e.g. code already taken)    | Yes, but rolls back | No — transaction rolls back, request stays `APPROVED` to retry |

### What a rejection does

One `REJECT` decision (by default, the first one) moves the whole request to `state = REJECTED`
**immediately** — even if it lands at step 2 of 3, even if others had already approved. That state is
**terminal**:

- **`apply` is never called**, so the domain table is never touched — no half-row, nothing to clean up.
- The maker gets a `rejected` notification. If they still want the change, they raise a **brand-new**
  request; you cannot un-reject or resurrect the old one. The rejected row stays in the audit log
  forever (BRH 10-year retention).

The threshold is configurable per policy (`rejectionsToReject`, default 1) if you want "two people must
object before it dies" — but approve-side and reject-side are deliberately asymmetric: saying no is
cheap.

---

## 4. Part C — The promises (what each side guarantees)

A contract is only useful if each side can rely on the other without reading its code. These are the
guarantees.

### The module promises the caller

1. **No effect before `APPLIED`.** A submitted change writes nothing in your domain. The pending
   instrument is not an instrument. Your `apply` is the *only* moment your rows are touched.
2. **`apply` is called at most once per request, and only after the pinned policy is fully satisfied**
   — separation of duties, eligibility, order, N-eyes and rejection rules all already enforced. By the
   time you are called, the decision is final.
3. **Atomicity.** `apply(payload)` and the flip to `APPLIED` commit in **one database transaction**. If
   your `apply` throws, the transaction rolls back: no domain rows, and the request stays `APPROVED`
   (not `APPLIED`) so it can be retried. You never end up half-written. This is why `APPROVED` exists as
   a distinct state you rarely see at rest.
4. **The payload comes back byte-for-byte** as you submitted it.
5. **Events fire** at `submitted` / `approved` / `rejected` (and application is observable via the state
   reaching `APPLIED`) so notifications and audit can react without you wiring them.

### The caller promises the module

1. **Call `submit_change` unconditionally** on every gated write path. Do not pre-check whether a gate
   exists — that is the module's decision (§6). Your job is to always ask.
2. **A registered applier** under your `targetType`, whose three methods honour their contracts —
   `validate`/`render` never write, `apply` performs the *complete* write and returns the id.
3. **A self-contained, versioned payload** (§2.2, §2.3). Everything `apply` needs is inside it.
4. **`apply` is idempotent-safe under retry.** Because a failed apply leaves the request retryable,
   `apply` may be invoked again. Guard against double-writes (natural key, upsert, or the expected-version
   check) so a retry after a partial external side-effect does not duplicate.
5. **Upstream permission is already checked.** Whether this maker is even allowed to *propose* the change
   is RBAC's concern, before `submit_change`. The module assumes the proposal itself was authorized.

---

## 5. The call flow, end to end

```mermaid
sequenceDiagram
    participant C as Caller (domain module)
    participant M as approbation
    participant R as Applier registry
    participant K as Checkers

    C->>M: submit_change(targetType, targetId, operation, payload, makerId)
    M->>R: look up applier by targetType
    M->>R: applier.validate(payload)   %% fail fast at submit
    alt no active policy for (targetType, operation)
        M-->>C: pass-through — applied immediately, no request row
    else policy exists
        M->>M: park ChangeRequest (SUBMITTED), pin policyId (banded by threshold)
        M-->>C: event: submitted
        K->>M: view request
        M->>R: applier.render(payload)  %% checker sees domain language
        K->>M: decisions (APPROVE / REJECT)
        M->>M: enforce SoD, eligibility, order, N-eyes; compute satisfaction
        alt policy satisfied
            M->>M: state → APPROVED
            Note over M,R: one database transaction
            M->>R: applier.validate(payload)  %% re-check at apply
            M->>R: applier.apply(payload) → id
            M->>M: stamp targetId, state → APPLIED
            M-->>C: event: approved (+ APPLIED observable)
        else a checker rejects
            M->>M: state → REJECTED (terminal)
            M-->>C: event: rejected
        end
    end
```

---

## 6. Gate-or-pass is the module's call, not yours

The caller does **not** branch on "is this gated?" It always calls `submit_change`. The module decides:

- **Active policy exists** for `(targetType, operation)` → the change is parked and gated.
- **No active policy** → the change passes straight through: `validate`, then `apply` immediately, no
  request row written at all.
- **A policy with zero steps** → "logged but auto-approved": a request row for the audit trail, applied
  without a human gate.

This is the point of the whole design: a flow moves *under* or *out of* maker-checker **by editing
policy data, with no change to the caller's code**. The caller wrote `submit_change` once and forgot
about it; an admin turns the gate on for `(instrument, CREATE)` a year later and creating instruments
starts requiring approval — the instrument module never redeploys.

---

## 7. Worked example — gating instrument creation

**Registration (once, at module load):**

```
approbation.register_applier(
    targetType = "instrument",
    applier    = InstrumentApplier(),   # implements validate / render / apply
)
```

**The gated write path (called every time, gate or no gate):**

```
submit_change(
    targetType = "instrument",
    targetId   = None,                  # CREATE → no id yet
    operation  = "CREATE",
    payload    = {
        "payloadVersion": 1,            # caller's schema version (§2.3)
        "code": "OBL-BRH-2029",
        "instrumentTypeId": "...",
        "assetClassId": "...",
        # ... everything apply() will need, self-contained
    },
    makerId    = current_user.id,
)
```

**The applier the caller implements:**

```
class InstrumentApplier:
    def validate(payload):   # no writes; called at submit and at apply
        assert payload["payloadVersion"] == 1
        assert Instrument.code_is_free(payload["code"])   # re-checked at apply
        ...

    def render(payload):     # domain language for the checker
        return f'Create instrument {payload["code"]} ({payload["instrumentTypeId"]})'

    def apply(payload):      # the only place instrument rows are written
        instrument = Instrument.create(**strip(payload, "payloadVersion"))
        return instrument.id     # stamped back into targetId
```

The maker-checker module ran this entire flow — parked it, resolved the policy, gathered two approvals,
enforced that the maker was not one of them — **without importing `Instrument` or knowing what a `code`
is.** That is the contract working.

---

## 8. Onboarding checklist for a new caller

To bring a module under maker-checker, the caller must:

- [ ] Choose a **stable, namespaced `targetType`** string and never rename it.
- [ ] Implement an **applier** — `validate` (no writes), `render`, `apply` (complete write, returns id).
- [ ] **Register** the applier under the `targetType` at startup.
- [ ] Make the **payload self-contained** and stamp a **`payloadVersion`**.
- [ ] For `UPDATE` / `DEACTIVATE`, carry an **expected-version token** and reject stale writes in `apply`.
- [ ] Call **`submit_change` unconditionally** on the gated write path — never pre-check for a gate.
- [ ] Make `apply` **safe to retry** (a failed apply leaves the request retryable).
- [ ] Leave **who-may-propose** to upstream RBAC.

The other side — policies, steps, eligibility, the rules — is configuration and belongs to admins, not
to the caller's code. That asymmetry is the payoff: **callers write an applier once; the gate is data
forever after.**

---

## 9. Anti-patterns (things that quietly break the contract)

- **Peeking into the payload from the module side.** If you ever find `approbation` reading a payload
  field by name, the seam has leaked and the module is no longer generic.
- **A `targetType` that encodes the operation** (`instrument_create`). Keep them orthogonal:
  `targetType` = entity, `operation` = verb. Policies band on the pair.
- **A payload that references request-scoped state gone by apply time.** Self-contained means
  self-contained.
- **`apply` that also validates business permission.** Permission is upstream; `apply` performs the
  write and trusts that approval already happened.
- **Writing anything domain-specific from inside `approbation`.** The module writes its own five tables
  and calls your `apply`. If it writes an instrument directly, delete that code — that is what the
  applier is *for*.
- **Pre-checking "is this gated?" in the caller.** It couples the caller to the gate's existence and
  defeats switch-by-data. Always `submit_change`; let the module decide.

---

*In one line:* the contract is an **opaque envelope in** (`targetType`, `targetId`, `operation`,
`payload`) plus a **registered applier back** (`validate` / `render` / `apply`) plus a set of
**promises** (no effect before `APPLIED`, one-transaction atomicity, byte-exact payload, lifecycle
events) — and the Django `(app_label, model, object_id)` you already knew is just the *addressing* corner
of the envelope.
