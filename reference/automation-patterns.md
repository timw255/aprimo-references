# Automation patterns: references + fields + rules

A reference computes **one value**. Many real requests are not "write a reference" but "make Aprimo
*do* something" — expire assets on a date, e-mail an owner before expiry, fire an HTTP call, keep a
field in sync. Those need machinery a single reference can't provide, and recognizing that is half
the job:

- A **reference** computes a value (the `ref:` DSL this skill verifies).
- A **field** holds state between runs (including flags the automation reads and sets).
- A **rule** decides **when** it happens (a trigger) and **what** happens (actions).

So when someone asks for "a reference to do X," check whether X actually needs a field and a rule.
If it does, **say so and lay out the recipe** — the fields to add, the rule (trigger / condition /
action), and the reference snippets that go inside it (which you can still write and verify here).

## Where references run

A reference is always *embedded* in one of these slots. What it must return depends on the slot:

| Slot | What it does | The reference returns |
|---|---|---|
| **Field default value** | computes the field's value when its "reset to default" trigger fires | any value valid for that field (e.g. `<ref:now/>`, a computed date, a string) |
| Rule action **Set field value** | writes the value into a **text or option-list** field (must already be on the item, or field scope = *Record Content Floating*) | the string to store |
| Rule action **Execute reference** | runs the reference **for its side effect** — *this is how a `ref:httpRequest` actually fires*; a reference sitting in a field default does not make the call | (the side effect; return value is ignored) |
| Rule action **Send e-mail** | the reference is the e-mail template | a `<mail>` structure (see below) |
| Rule condition **Reference match** | gates the rule on a boolean — only with the *save/delete* trigger; users who trigger the rule need **read** permission on the fields the reference reads | `True` / `False` (use `ref:compare`) |
| Rule action **Link / Unlink classification** | dynamic classification targets | classification Id(s) or name path(s), comma-separated (e.g. a `ref:foreach` over a Classification List field) |
| **File-naming convention** | names a generated rendition / download file | the file name — see [`../docs/file-naming.md`](../docs/file-naming.md) |

`out` values are lowercase at runtime (`out="id"`, not `out="Id"`) — verify with `execute`.

## References are not rule conditions

A rule's **condition** is *not* a reference — it's a comparison expression typed into the rule
builder, and the bundled compiler does **not** validate it:

```
Field <operator> value
```

- **Date / DateTime fields:** operator `=`, `<=`, `>=`; the value can only be `Now`, `Now+n`, or
  `Now-n`, where `n` is a number of **days**. E.g. `ExpirationDate <= Now`, `ExpirationDate <= Now+30`.
- **Option-list fields:** only `=` (a multi-value match is enough). E.g. `Status = "Active"`.
- **Text fields:** `=`, `<>`, `>`, `>=`, `<`, `<=`, `Contains`, `In`. E.g. `Description Contains "draft"`.

References (the `ref:` DSL) are used in **field defaults** and **rule actions** — not in these
conditions. The one exception is **Reference match**, below.

### Reference match — when a reference *is* the condition

The **Reference match** condition (available only on the *save/delete* trigger) fires the rule when
the given reference returns **`True`**. This is the escape hatch for conditions the simple
`Compare field value` syntax can't express — comparing **two fields to each other**, combining
several checks, substring/regex matches, or date math beyond `Now±n`. Build the boolean with
`ref:compare` (and gates for AND/OR — see [patterns.md](patterns.md)), make the reference output
`True`/`False`, and verify it like any other reference:

```xml
<ref:record fieldName="ApprovedBy" out="value" store="@approver"/>
<ref:record fieldName="CreatedBy"  out="value" store="@creator"/>
<ref:compare value1="@approver" value2="@creator" operator="ne"/>
<!-- True when the approver is not the creator (separation of duties) -->
```

The rule then runs its actions on exactly the items where this comes back `True`. (Permission note:
users who trigger the rule need **read** access to every field the reference reads, or it won't fire.)

## Triggers, and the nightly constraint

A rule's trigger ("Check rule conditions") is one of:

- **when target is saved or deleted** — fires on the event. Supports all condition and action types,
  and each action can run *immediately* (before save/validation) or *delayed* (in a maintenance job).
- **on schedule (daily)** — checks the conditions on a fixed time **every night**. Use this for
  date-based logic. **Only the `Classified in` and `Compare field value` conditions are supported,
  and all actions run immediately** (no immediate/delayed choice).

Two more: **Send e-mail** is *delayed-only*, and rule actions that contain **HTTP-request references**
should be set to *delayed* (immediate execution of those is deprecated).

## Worked recipe: asset expiration

This is the canonical *fields + scheduled rule + state flag* shape.

**Fields to add:**
- `ExpirationDate` — a Date or DateTime field holding the target date (can itself be defaulted by a
  reference, e.g. upload date + 5 years via `ref:datetimeeval`).
- `IsNotificationSent` — a read-only Text field, **default value `No`** (the re-fire guard).

(For *how* to configure these — which recompute trigger a defaulted date needs, why the compared
field must be **stored** for the nightly search to find it, scope, and reaching a related record's
value — see [field-configuration.md](field-configuration.md).)

**Rule 1 — mark expired on the date.** Target *Record / File*; trigger *on schedule (daily)*.
- *If:* `ExpirationDate <= Now` (add `Status = "Active"` or a `Classified in` to narrow the set).
- *Then:* one or more of — **Change status to** archived · **Set field value** (e.g. a `HasExpired`
  flag, via a reference returning `Yes`) · **Link / Unlink classification** (move to a restricted
  branch) · **Apply a watermark**.

**Rule 2 — warn 30 days before.** Target *Record / File*; trigger *on schedule (daily)*.
- *If:* `ExpirationDate <= Now+30` **AND** `IsNotificationSent = "No"`.
- *Then:* **Send e-mail** (the template below) · **Set field value** → set `IsNotificationSent` to
  `Yes`. Because the next nightly run requires `IsNotificationSent = "No"`, the warning is sent once.

## The notification e-mail template

The **Send e-mail** action takes a reference shaped as a `<mail>` document. The `<mail>`,
`<recipients>`, `<cc>`, `<subject>`, `<body>` wrappers are the action's structure; the **`ref:` tags
inside each slot are ordinary references** you can write and verify here:

```xml
<mail>
  <recipients>
    <ref:record fieldName="AssetOwner" out="value" store="@OwnerId"/>
    <ref:user id="@OwnerId" out="email"/>
  </recipients>
  <cc>
    <ref:setting name=".technicalContactEmail"/>
  </cc>
  <subject>
    <ref:text out="This record will soon expire"/>
  </subject>
  <body>
    <ref:object>
      <ref:text out="The record with id "/><ref:record out="id"/><ref:text out=" will expire within 30 days."/>
    </ref:object>
  </body>
</mail>
```

Verify each slot's reference on its own (pipe just the `<recipients>`/`<subject>`/`<body>` contents
through `execute`) — don't pipe the `<mail>` wrapper; it isn't a reference. You can also keep the
whole template in a custom setting and point the action at it with `<ref:setting name="YourSetting"/>`.

## Adaptable use cases

Same shape — a field or two, a scheduled `Compare field value` condition, and an action:

- **Auto-archive / retention** — `RetentionDate` + a `flagged`/`notified` guard; nightly
  `RetentionDate <= Now` → Change status / Link to a "For deletion" branch.
- **Review-due reminders** — `ReviewDueDate` + `ReviewReminderSent` (default `No`); nightly
  `ReviewDueDate <= Now+7` AND `ReviewReminderSent = "No"` → Send e-mail + set the flag.
- **License / usage-rights expiry** — `LicenseExpiry` + alert guard; nightly `LicenseExpiry <= Now+30`
  → Send e-mail / Unlink from "published".
- **Scheduled status transitions** — a DateTime field + the **Schedule a resave of the record** action
  (resaves the item on that date, which then triggers your save-based rule).
- **Stale-asset flagging** — `LastModified` + `IsStale`; nightly `LastModified <= Now-90` → Set
  `IsStale = Yes` / Link to a "Needs review" branch.

## What the compiler verifies — and what it doesn't

- **It verifies the `ref:` expressions:** field-default references, Set-field-value references, the
  individual e-mail slots, and `Reference match` `ref:compare` expressions. Lint and execute those as
  always.
- **It does not verify:** the `<mail>` wrapper, the `Field <= Now+n` condition text, or the
  field/rule configuration itself — those are set up in the DAM admin UI.

So the deliverable for a "make X happen" request is usually: **(1)** the field(s) to add, **(2)** the
rule — trigger, conditions, actions, and **(3)** the verified reference snippets that go inside it.

## See also
- [`field-configuration.md`](field-configuration.md) — field types, default triggers, scope, storage, and Record-Link traversal behind these recipes.
- [`../docs/httprequest.md`](../docs/httprequest.md) — the HTTP-request reference fired by the *Execute reference* action.
- [`../docs/setting.md`](../docs/setting.md) — reading settings (e.g. a shared e-mail template or contact address).
- [`../docs/datetimeeval.md`](../docs/datetimeeval.md) — computing the dates these recipes compare against.
- [`../docs/file-naming.md`](../docs/file-naming.md) — references as file-naming templates.
- [`patterns.md`](patterns.md) — reference-expression idioms (gating, pick-a-value, crash-safety).
