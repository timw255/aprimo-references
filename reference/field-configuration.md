# Field configuration (types, defaults, triggers, scope)

References *read* fields ([`../docs/field.md`](../docs/field.md)) and *compute* their defaults, and
rules *set* and *compare* them. But whether a default recomputes, whether a scheduled rule can even
see a field, and whether a rule may set it are all decided by how the **field definition** is
configured. This page documents the configuration that matters when you build functionality
(expiration, notifications, derived values, cross-record data). It's admin-side setup — the bundled
compiler can verify the reference in a default or action, but not the field settings themselves.

## Default values, and when they recompute

A field's default value can be a **reference** (e.g. `<ref:now/>`, or a computed date). The field's
**Reset to Default Triggers** decide *when* that default recalculates:

| Trigger | The default recomputes… |
|---|---|
| **None** | only when the field is first initialized, then never again |
| **On New Field** | when the field is added to an object (the default) |
| **On Save** | every time the object is saved |
| **On Load** | every time the object is opened/loaded |
| **On Any Change** | on any change to the object |
| **On Field Change** | when a change is made to **specific fields you choose** (see below) |
| **On Status Change** | when the record's status changes — **not** when status is set at creation |
| **On Any File Change** / **On Master File Change** / **On Current File Change** | on file changes (record/file scope) |
| **On Reclassify Record** | when the record is linked to / unlinked from a classification |
| **On Field Definition Change** | when the field definition itself changes |
| **On Duplicate Record** | when the record is created as a duplicate |

- **`On Field Change` watches specific fields.** Pick the watched fields in **Reset to Default
  Fields**; the default recomputes whenever any of them changes. This is how a *derived* field stays
  current — e.g. a "warn date" defaulted to `expiration − 30 days` recomputes when `ExpirationDate`
  changes.
- **A default only recomputes on *this* record's triggers.** There is **no trigger that fires when a
  *related* record changes.** So a default that reads a linked record's value goes **stale** when the
  linked record changes — see [Crossing records](#crossing-records) for the durable alternative.

A **read-only** field with a default reference is the standard "computed field" — the value is always
calculated, never typed. The state flags automations rely on (e.g. `IsNotificationSent`, default
`No`) are exactly this: read-only, defaulted to a literal.

## Scope — which records get the field

Scope decides which objects a field is added to. It matters because **rules only set fields that are
already on the item** (the *Set field value* action auto-adds a missing field **only** if its scope is
*Record Content – Floating*), and a **scheduled rule can only act on records that actually carry the
field it compares**.

| Scope | The field is on… |
|---|---|
| **Record Content – Global** | every record |
| **Record Content – Class Dependent** | records in a classification (or of a content type) where the field is registered |
| **Record – Content Type Dependent** | records of a specific content type |
| **Record Content – Floating** | only records it's manually added to (rules can auto-add it) |
| **Classification Profile – Class/Floating** | classifications (registered or manual) |

For a field a scheduled rule must reliably find (a date, a flag), prefer **Global**,
**Class Dependent**, or **Content Type Dependent** so the field is guaranteed present.

## Storage mode — searchable or not

A scheduled rule gathers its records by building a **search expression** over field values, so any
field its condition compares must be **stored**:

- **Values are never stored** — calculated default only, **not searchable** → a scheduled rule's
  `Compare field value` will never match it. Avoid for any field a rule compares.
- **Non-empty values are stored** (default) — stored and searchable unless empty.
- **All values are stored** — stored even when empty; can also **log all field value changes** (audit).

This is the most common silent reason a scheduled rule "does nothing": the compared field isn't stored.

## What rules can set vs. read

The **Set field value** rule action writes a reference's result into **Text or Option List fields
only** — not Date, Number, User/User List, or Classification List. So:

- Flags and text labels → set with **Set field value**.
- A **date/number** value → you can't set it with a rule action; compute it as a **default** instead.
- A **User** field → populate via a default (`<ref:user out="id"/>`), not a rule.
- **Classifications** → use the **Link / Unlink classification** rule actions, not Set field value.

**Validation routines are read-only.** A field's validation reference can only *check* a value
(returning, via `ref:compare`, whether it's valid) — it can **never set** a field. Use a rule to
change values. (Validation trigger **Always** is recommended for a field whose validity depends on
another field; `field="current"` reads the field being validated.)

## Field types worth knowing

| Type | Notes for automation |
|---|---|
| **Text** | settable by a rule; supports regex validation and min/max length; HTML/script is sanitized on save |
| **Option List** | settable by a rule; options can be **filtered** by a reference (e.g. limit by another field's value) |
| **Date** vs **DateTime** | Date = day/month/year; DateTime = date **and** time. Read parts with `out="year"/"month"/"day"`; DateTime supports `out="localvalue"/"utcvalue"` |
| **Number / Text (unique)** | can enforce a **unique identifier** across the system (rebuild the unique-id cache after toggling) |
| **User / User List** | not settable by a rule; populate via default; can be **filtered** by a reference (e.g. by group) |
| **Classification List** | not settable by *Set field value*; managed via Link/Unlink actions; can be filtered, and can sync to the record's actual classification links |
| **Record Link** | relates records — see [Crossing records](#crossing-records) |

**Use UTC for dates that mark a moment** (expiration, deadlines): set the field's *Use UTC* option so
`Now` comparisons and `out="utcvalue"` line up across time zones.

When several defaults depend on each other, **Sort Index** sets the calculation order — lower index
computes first.

## Crossing records

A **Record Link** field connects records, and the link is **bidirectional**: linking A→B also links
B→A, so you can navigate from either side.

**Read related records** with these field projections. The names are **inverted** — this is
documented behavior, not a typo:

| `out` | Returns |
|---|---|
| `valuechildren` | the **parent** records' values |
| `valueparents` | the **child** records' values |
| `valuelinks` | the linked records' values |

Each returns a collection of related records — iterate and read each by `id`:

```xml
<ref:record fieldName="Finals" out="valuelinks" store="@finals"/>
<ref:foreach in="@finals" storeitem="@finalId" join=";">
  <ref:record id="@finalId" fieldName="Owner" out="value" store="@ownerId"/>
  <ref:user id="@ownerId" out="email"/>
</ref:foreach>
```

To put a **related record's value onto this record** (so a scheduled rule can compare it), you have
two options:

1. **A default that reads the linked record** — simple, but it recomputes only on *this* record's
   triggers, so it goes **stale** when the linked record changes (there is no related-record trigger).
2. **Metadata Inheritance** — a child inherits designated fields from a parent **read-only**, and
   *"each time the field on the parent is changed, the value is propagated to the child content
   items"* (in the background). This is the durable way to bring, say, an expiration date down from a
   linked record onto the record a rule runs on. (Inherited fields are separate from the record's own;
   confirm they're searchable by scheduled rules in your tenant.)

So for a goal where the trigger value (a date) and the target (an owner) live on different related
records, either **inherit the date onto the junction record and run the rule there**, or **run the
rule where the date lives and traverse the links in the action** (recipients/Link-classification).
See [automation-patterns.md](automation-patterns.md) for the end-to-end recipes.

## See also
- [`../docs/field.md`](../docs/field.md) — reading field values with references (the `out` projections).
- [`automation-patterns.md`](automation-patterns.md) — the fields + rules + scheduled-check recipes these settings enable.
- [`../docs/datetimeeval.md`](../docs/datetimeeval.md) — computing date defaults (e.g. upload + 5 years).
