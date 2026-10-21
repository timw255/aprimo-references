# Object reference (`ref:object`)

Groups references and emits a value, **preserving the underlying object/type** instead of coercing it to text. It is the type-aware sibling of [`ref:text`](text.md): same grouping and gating, but use `ref:object` when the result is a real object (a date, a collection, a record link) that a later reference will consume rather than print. It is a **container tag**.

## Syntax
```xml
<ref:object out="value"/>
```
or, with nested children:
```xml
<ref:object> ...child references... </ref:object>
```

## Attributes
| Attribute | Description |
|---|---|
| `out` | The value (or `@variable`) to output. |
| `name` | Optional object name (object-specific attribute). |
| `store` | Capture the value into a variable **without emitting** it. |
| `onVariable` | Gate output on a predicate: `IsNotEmpty(@val)`, `IsEmpty(@val)`, `IsNotZero(@val)`, `IsZero(@val)`. |
| `onAllVariables` / `onAnyVariable` | Gate on whether listed variables are **defined / non-null** (AND / OR). |
| `format` / `join` | Format the value / join a collection. |

## Examples

**Conditionally group output.** The block emits only when the gate passes — the standard way to assemble multi-part output under a single condition:
```xml
<ref:record out="status" store="@status"/>
<ref:object onVariable="IsNotEmpty(@status)">
    <ref:text out="Status: "/>
    <ref:text out="@status"/>
</ref:object>
```
Produces: `"Status: Approved"` (context: `{"records":[{"status":"Approved"}]}`). If `@status` were empty the whole block emits nothing.

**An opposite gate emits nothing** when its condition is false:
```xml
<ref:record out="status" store="@status"/>
<ref:object onVariable="IsEmpty(@status)">
    <ref:text out="no status"/>
</ref:object>
```
Produces: `""` (status is present, so the `IsEmpty` gate fails)

**Hold a computed date as an object for downstream use.** Storing through `ref:object` keeps the date as a real value you can compare or re-emit:
```xml
<ref:now store="@nw"/>
<ref:datetimeeval in="@nw" addYears="5" store="@future"/>
<ref:object out="@future" store="@dt"/>
<ref:text out="@dt"/>
```
Produces: e.g. `"2031-06-15T13:58:16-04:00"` (now + 5 years)

**Emit a literal** (behaves like `ref:text` for plain strings):
```xml
<ref:object out="hi"/>
```
Produces: `"hi"`

## Raw HTML / XML markup inside a container

Elements that are **not** `ref:` tags - `<b>`, `<a href="...">`, `<div>`, `<br/>`, `<p>` - are
allowed inside `ref:object` (and inside `ref:catch`, `ref:foreach` bodies and `<item>`s) and pass
through **literally** to the output. This is how e-mail bodies are built:

```xml
<ref:application out="damUrl" store="@damUrl"/>
<ref:record out="id" store="@id"/>
<ref:object><a href="@damUrl/contentitems/@id"><ref:record out="title"/></a><br/><div>Please review.</div></ref:object>
```
Produces: `<a href="https://tenant.dam.aprimo.com/dam/contentitems/...">Quarterly Report</a><br/><div>Please review.</div>`

Two rules: `@variables` are substituted inside the markup's **attributes** (`href="@damUrl/..."`)
but **not** in its text content - emit text through `<ref:text out="@var"/>`; and the markup
must be well-formed XML (`<br/>`, not `<br>`; `&amp;` for a literal `&`).

## Gotchas

- **`object` vs `text`:** reach for `ref:text` for human-readable string output; reach for `ref:object` when a later reference must consume the **typed** value (a date object, a collection) rather than its string form. For plain literals they behave the same.
- **Same gating semantics as `text`.** `onAllVariables`/`onAnyVariable` test whether each variable is **defined (non-null)**, not non-emptiness or truth — a variable stored as `""` passes, and a false `compare` result (`"False"`) passes; only a *never-stored* variable is false (it does not error). Use `IsNotEmpty` for "is it populated" and `IsNotZero`/`IsZero` for real truth tests.
- **`store` suppresses output**, just like on `text`.
- **Close it with `</ref:object>`.** A mismatched `</object>` is a parse error.

## See also
- [`text.md`](text.md) — string-coercing grouping and the full gating reference.
- [`../reference/patterns.md`](../reference/patterns.md) — conditional defaults built from nested gated `object` blocks.
- [`../reference/operators-and-logic.md`](../reference/operators-and-logic.md) — predicates and combining conditions.
