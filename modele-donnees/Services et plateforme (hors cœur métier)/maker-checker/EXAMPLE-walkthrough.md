# Maker-Checker by example — a plain walkthrough

This file exists to make [CONTRACT.md](CONTRACT.md) *feel* obvious. No abstract concepts. We follow one
real instrument being registered, and we look at the **actual database rows** at every step so you can
see exactly where things live and when they change.

The cast:

- **Marie** — a back-office operator. She wants to register a new bond. She is the **maker**.
- **Paul** and **Jacques** — supervisors who approve sensitive changes. They are the **checkers**.
- The admin has configured one rule: **creating an instrument needs 2 approvals.**

---

## The single most important idea, up front

> **A proposed instrument is NOT an instrument. It is a piece of paper describing an instrument.**

Until it is approved, the thing Marie created does **not** live in the `instrument` table. It lives as
JSON inside a `change_request` row, in the approbation module. The `instrument` table stays empty. It
gets its first row only at the moment Paul gives the second approval.

Hold onto that. Everything below is just that sentence, in slow motion.

---

## Step 0 — Before anything happens

Two tables matter. Here they are, empty:

**`instrument`** (owned by the instrument module)

| id | code | type | status |
| -- | ---- | ---- | ------ |
| *(empty)* | | | |

**`change_request`** (owned by the approbation module)

| id | targetType | targetId | operation | payload | state | makerId |
| -- | ---------- | -------- | --------- | ------- | ----- | ------- |
| *(empty)* | | | | | | |

Note the `instrument` table does **not** have a "PENDING" column and never will. That is on purpose.

---

## Step 1 — Marie fills in the form and clicks "Register"

Marie types in the new bond: code `OBL-BRH-2029`, type `BOND`, and so on. She clicks Register.

Here is the part that trips everyone up the first time: **her click does not write to the `instrument`
table.** Instead, the instrument module's "create instrument" code does this — *every single time,
whether or not approval is switched on:*

```
submit_change(
    targetType = "instrument",     # what kind of thing
    targetId   = None,             # it doesn't exist yet — this is a CREATE
    operation  = "CREATE",
    payload    = {                 # the "piece of paper" — just data
        "code": "OBL-BRH-2029",
        "type": "BOND",
        ... everything needed to actually create it later ...
    },
    makerId    = marie.id,
)
```

The approbation module receives this, sees that a rule exists for `(instrument, CREATE)`, and **parks
the proposal**. Now the tables look like this:

**`instrument`** — still empty. Marie's bond is NOT here.

| id | code | type | status |
| -- | ---- | ---- | ------ |
| *(empty)* | | | |

**`change_request`**

| id | targetType | targetId | operation | payload | state | makerId |
| ---- | ---------- | -------- | --------- | ------------------------ | ----------- | ------- |
| CR-1 | instrument | *(null)* | CREATE | `{code: OBL-BRH-2029, …}` | **SUBMITTED** | Marie |

**This is the answer to your question.** The proposed instrument is `CR-1.payload` — a blob of JSON.
It is not an instrument row. `targetId` is empty because the instrument does not exist yet; there is
nothing to point at.

Marie's screen shows: *"Your request to create OBL-BRH-2029 is waiting for approval."* She now has a
thing to watch: **CR-1**.

---

## Step 2 — Paul opens his approval inbox

Paul sees a pending request. But remember — the approbation module has **no idea what an instrument
is.** It can't show Paul a nice instrument screen. So it asks the instrument module to do it, by calling
the instrument module's `render(payload)`:

> **Request CR-1 — Marie proposes: Create bond instrument "OBL-BRH-2029" (type BOND, matures 2029).**

Paul reads it, agrees, and clicks Approve. The module records his decision:

**`approval_decision`**

| id | changeRequestId | approverId | decision |
| ---- | --------------- | ---------- | -------- |
| D-1 | CR-1 | Paul | APPROVE |

The rule needs **2** approvals. We have 1. So `CR-1` is still `SUBMITTED`. **The `instrument` table is
still empty.** Nothing has happened to the domain yet.

---

## Step 3 — Jacques gives the second approval → the magic moment

Jacques opens CR-1, reads the same rendered summary, and clicks Approve.

**`approval_decision`**

| id | changeRequestId | approverId | decision |
| ---- | --------------- | ---------- | -------- |
| D-1 | CR-1 | Paul | APPROVE |
| D-2 | CR-1 | Jacques | APPROVE |

Now the rule is satisfied (2 of 2). **This is the moment the instrument gets created.** In **one single
database transaction**, the approbation module does two things that both succeed or both fail together:

1. It calls the instrument module's `apply(payload)` — *now, finally,* the instrument module writes its
   own table:

   **`instrument`**

   | id | code | type | status |
   | ---- | ------------ | ---- | ------ |
   | **INS-42** | OBL-BRH-2029 | BOND | ACTIVE |

   `apply` returns the new id: **INS-42**.

2. It updates the change request: state → `APPLIED`, and stamps the returned id into `targetId`:

   **`change_request`**

   | id | targetType | targetId | operation | state | makerId |
   | ---- | ---------- | ---------- | --------- | ----------- | ------- |
   | CR-1 | instrument | **INS-42** | CREATE | **APPLIED** | Marie |

Because both happen in one transaction: if `apply` had crashed halfway (say the code was already taken),
the instrument row would be rolled back *and* the request would stay `APPROVED` (not `APPLIED`) so it can
be retried. You can never end up with a half-created instrument.

---

## Step 4 — How does Marie find out?

This is your other question. Two ways, and they work together:

1. **She watches CR-1.** The request she created now reads `state = APPLIED` and `targetId = INS-42`.
   That is her proof it worked, and her handle to the real instrument.
2. **She gets a notification.** When the request flipped, the module emitted an `approved` event. The
   notifications module was listening, so Marie gets: *"Your instrument OBL-BRH-2029 was approved and
   created."* She never had to sit and refresh a page.

Done. The instrument exists, it's real, and every row in the `instrument` table is a genuine approved
instrument — there was never a fake "pending" one sitting in there.

---

## The same story when a checker says no

Rewind to Step 3, but Jacques clicks **Reject** instead:

**`change_request`**

| id | targetType | targetId | operation | state | makerId |
| ---- | ---------- | -------- | --------- | ------------ | ------- |
| CR-1 | instrument | *(null)* | CREATE | **REJECTED** | Marie |

That's the whole effect. `apply` is **never called**, so the `instrument` table is **never touched** —
it stays empty. There is no rejected instrument to clean up, because there was never an instrument. Marie
gets a "your request was rejected" notification and, if she still wants the bond, she raises a fresh
request. `REJECTED` is final; you don't un-reject.

*(This is the big win over the "save it with a PENDING status" approach: there, a rejection leaves a dead
row in your real table that you have to hide from every query forever. Here, there's simply nothing.)*

---

## A second example — editing an existing portfolio (UPDATE)

CREATE is the easy case because the thing doesn't exist yet. UPDATE has one extra wrinkle worth seeing.

Marie wants to rename portfolio **P-7** from "Fonds Alpha" to "Fonds Alpha Croissance". Same flow:

```
submit_change(
    targetType = "portfolio",
    targetId   = "P-7",          # <-- this time it DOES exist, so we name it
    operation  = "UPDATE",
    payload    = { "name": "Fonds Alpha Croissance", "expectedVersion": 5 },
    makerId    = marie.id,
)
```

**During approval, the live portfolio P-7 is unchanged.** Its name in the `portfolio` table is still
"Fonds Alpha". Anyone looking at P-7 sees the old name. The proposed new name lives only in the
`change_request.payload` — exactly like the instrument did. The edit takes effect only when `apply` runs
on final approval.

The wrinkle: what if, *while Marie's rename waits for approval*, someone else already renamed P-7 to
something else? Then Marie's approved change would silently overwrite their newer change. That's why the
payload carries `expectedVersion: 5` — a note saying "I based this on version 5 of the portfolio." When
`apply` runs, the portfolio module checks: *is P-7 still at version 5?* If yes, apply. If it's now at
version 6, **refuse** and let the request fail, so Marie is told "the portfolio changed under you, please
redo." This is the portfolio module's job (it owns the portfolio), not approbation's — approbation can't
even read the payload. It's the same class of protection the position module uses to kill the old "Axia
desync" bug.

You don't need this for CREATE (nothing exists to collide with), which is why the instrument example
didn't mention it.

---

## The mental model, boiled down

| Question | Answer |
| --- | --- |
| Where does a *proposed* instrument live? | As JSON in `change_request.payload`. **Not** in the `instrument` table. |
| Does the `instrument` table have a PENDING status? | **No.** It only ever holds real, approved instruments. |
| When is the `instrument` row actually written? | At the moment of final approval, inside `apply()`, in one transaction. |
| What if it's rejected? | `apply` never runs; the `instrument` table is never touched; nothing to clean up. |
| How does the maker know it's done? | The `change_request` flips to `APPLIED` (and `targetId` gets the new id), **and** she gets a notification. |
| Who actually writes the instrument row? | The instrument module's own `apply()` — approbation just calls it. |
| Does the instrument module have to check "is approval on?" | No. It always calls `submit_change`. If no rule exists, the change just applies immediately. |

If you can retell the instrument story — *"the proposed bond is only a piece of paper in the approval
table until the second approval, and only then does a real instrument row appear"* — you can defend the
whole design.
