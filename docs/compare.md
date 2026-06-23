# Compare (`ref:compare`)

Compares two values with an operator and resolves to the string **`True`** (true) or **`False`** (false) — **not** `1`/`0`. This is the building block for every condition: gate output with `IsNotZero(@result)` (true for `True`) / `IsZero(@result)` (true for `False`). Do **not** try to combine results with `ref:eval` — `ref:eval` is numeric-only and feeding it a `True`/`False` result errors ("invalid number"); combine conditions with nested gates instead (see [../reference/operators-and-logic.md](../reference/operators-and-logic.md)).

## Syntax
```xml
<ref:compare value1="value" value2="value" operator="operator" store="@result"/>
```

Values can be literals or `@variables`:

```xml
<ref:record fieldName="Status" out="value" store="@status"/>
<ref:compare value1="@status" value2="Approved" operator="eq" store="@isApproved"/>
```

## Attributes
| Attribute | Description |
|---|---|
| value1 | First value (literal or `@variable`). |
| value2 | Second value (literal or `@variable`). |
| operator | Comparison operator (closed set below). Defaults to `equal` if omitted. |
| store | Variable to capture the `True`/`False` result. With `store`, the compare emits nothing. |

### Operators (closed set — verified against the compiler)
| Meaning | Accepted spellings |
|---|---|
| equal | `equal`, `eq`, `==` |
| not equal | `notEqual`, `ne`, `!=` |
| greater than | `greaterThan`, `gt` |
| greater than or equal | `greaterThanOrEquals`, `ge` |
| less than | `lessThan`, `lt` |
| less than or equal | `lessThanOrEquals`, `le` |

That is the **entire** set. There is **no** `contains`, `startsWith`, `in`, `between`, `matches`, or `notContains`. The only symbol forms are `==` and `!=` — `>`, `>=`, `<`, `<=` are rejected, and it is `ge`/`le`, **not** `gte`/`lte`. An invalid operator fails lint with a message that lists every valid spelling.

### How ordering operators decide their comparison type (verified)
The ordering operators (`gt`/`ge`/`lt`/`le`) pick a comparison type from the operands:

- **Both values numeric → numeric comparison.** `gt("2","10")` is `False` (2 < 10 numerically).
- **Else both values parse as dates → datetime comparison.**
- **Otherwise → ordinal/lexicographic string comparison.** Text *does* order — there is no "ordering only works on numbers" limit; mixed/text comparisons are ordinal. Verified: `gt("Banana","Apple")` is `True`, `lt("Apple","Banana")` is `True`, and `gt("abc","5")` is `True` (text, because `"abc"` isn't a number).

`eq`/`ne` compare as **strings** (with date-equality when both parse as dates).

## Examples

**True/false gate** — branch on whether the master file is a JPEG:
```xml
<ref:record file="master" out="extension" store="@ext"/>
<ref:compare value1="@ext" value2="jpg" operator="eq" store="@isJpg"/>
<ref:text onVariable="IsNotZero(@isJpg)" out="image"/>
<ref:text onVariable="IsZero(@isJpg)" out="other"/>
```
Produces: `"image"` (with `-context-json '{"records":[{"files":{"master":{"extension":"jpg"}}}]}'`)

**Numeric compare on a file size** — flag files over 10 MB:
```xml
<ref:record file="master" out="filesize" store="@sz"/>
<ref:compare value1="@sz" value2="10485760" operator="gt" store="@big"/>
<ref:text onVariable="IsNotZero(@big)" out="LARGE"/>
<ref:text onVariable="IsZero(@big)" out="OK"/>
```
Produces: `"LARGE"` (with `-context-json '{"records":[{"files":{"master":{"filesize":12000000}}}]}'`)

**Date before/after** — compares date strings directly, no `datediff` needed:
```xml
<ref:compare value1="2026-12-31" value2="2025-01-01" operator="gt" store="@future"/>
<ref:text out="@future"/>
```
Produces: `"True"`

**Multi-condition AND via nested gates** — emit only if status is Approved AND value > 50:
```xml
<ref:compare value1="Approved" value2="Approved" operator="eq" store="@condA"/>
<ref:compare value1="100" value2="50" operator="gt" store="@condB"/>
<ref:object onVariable="IsNotZero(@condA)">
  <ref:text onVariable="IsNotZero(@condB)" out="VALID"/>
</ref:object>
```
Produces: `"VALID"`

**Default operator is `equal`** — omit `operator` for an equality check:
```xml
<ref:compare value1="x" value2="x" store="@cmp"/>
<ref:text out="@cmp"/>
```
Produces: `"True"`

## Gotchas
- **`store` suppresses output.** A `compare` with `store="@cmp"` prints nothing — it only writes the variable. The output you see comes from a later ungated `ref:text`.
- **A false compare still stores `"False"` (a defined, non-null value).** So `onAllVariables="@condA; @condB"` over two compare results is true whenever both *ran* — it checks that each variable is **defined**, not its truth. For "both conditions true," use nested `IsNotZero` gates.
- **Empty input fails gracefully.** `compare value1="" value2="2026-12-31" operator="gt"` returns `"False"` — it does not crash. You can run every compare unconditionally and they'll all be defined before their gates.
- **Whitespace breaks numeric compares.** A padded value like `" 100 "` silently fails a numeric compare (returns `False`). Strip it first with `ref:replace` (see [replace.md](replace.md)).
- **No substring/range operators.** For "contains," use `ref:regex`; for "is one of a list," do one compare per candidate and OR the gated results, or use `ref:switch`.

See also: [../reference/operators-and-logic.md](../reference/operators-and-logic.md) and [../reference/patterns.md](../reference/patterns.md).
