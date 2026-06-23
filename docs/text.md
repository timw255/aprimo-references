# Text reference (`ref:text`)

Emits a literal value, and — as a **container tag** — groups other references into a single, optionally-gated block. `ref:text` is the workhorse of output: it is the **only** thing that actually prints. Everything else either feeds variables (via `store`) or is grouped inside a `text`. If your rule produces no output, you are almost always missing an ungated `ref:text`.

## Syntax
```xml
<ref:text out="literal value"/>
```
or, with literal content / nested children:
```xml
<ref:text>literal value</ref:text>
<ref:text> ...child references... </ref:text>
```

## Attributes
| Attribute | Description |
|---|---|
| `out` | The literal value (or `@variable`) to output. |
| `store` | Capture the value into a variable **without emitting** it. |
| `onVariable` | Gate output on a predicate: `IsNotEmpty(@val)`, `IsEmpty(@val)`, `IsNotZero(@val)`, `IsZero(@val)`. |
| `onAllVariables` / `onAnyVariable` | Gate on whether listed variables are **defined / non-null** (AND / OR). See Gotchas. |
| `case` | `case="upper"` or `case="lower"` to change the value's case. |
| `left` / `right` / `padding` | Fit the value into a fixed width. See "Formatting attributes". |
| `format` | A .NET date format on a date value, or `MB`/`KB`/`GB`/`size` on a file size. See "Formatting attributes". |
| `encode` | Escape the value for embedding: `html`, `xml`, or `json`. |
| `join` | Separator used when the value is a collection. |

## Formatting attributes

These post-process the emitted value (and apply to most references, not just `text`):

- **`case`** — `case="upper"` / `case="lower"`. (To change case, use this — *not* `format`.)
- **`left="N"` / `right="N"`** — fit the value into a field exactly `N` characters wide. If it is longer it is truncated (`left` keeps the first `N`, `right` keeps the last `N`); if it is shorter it is padded (`left` pads on the right, `right` pads on the left) with `padding` (default a space). `N` must be `1` or more — a `0` or negative width throws (a catchable exception; an uncaught one yields no output). Both may be set: `left` runs first, then `right`. Examples: `left="3"` on `"Hello"` → `"Hel"`; `left="5" padding="0"` on `"Hi"` → `"Hi000"`; `right="3"` on `"Hello"` → `"llo"`.
- **`format`** — applies a [.NET date format](datetime.md) to a **date** value (`format="yyyy-MM-dd"`), or `MB`/`KB`/`GB`/`size` to a **file-size** value (`<ref:record file="master" out="filesize" format="size"/>` → `"1.73 MB"`). It does not change case and has no `number`/`uppercase`/`lowercase` values.
- **`encode`** — `html`, `xml`, or `json`.
  - `html` — the five markup characters become entities (`"`→`&quot;`, `'`→`&#39;`, `&`→`&amp;`, `<`→`&lt;`, `>`→`&gt;`); Latin-1 characters 160–255 (`é`, `ñ`, …) and astral characters (emoji) become numeric references `&#NNN;`. Other characters — including BMP symbols above 255 like `€` and `•` — stay literal.
  - `xml` — escapes only the five markup characters (`&`, `<`, `>`, `"`, `'`); leaves non-ASCII literal.
  - `json` — a JSON string including its surrounding quotes; the markup characters, the backslash, control characters, and **all** non-ASCII are escaped as `\uXXXX` (uppercase) over UTF-16 code units (so a Latin-1 character like `é` is escaped to its four-hex-digit `\uXXXX` form, and an astral character such as an emoji becomes a surrogate pair of two `\uXXXX` escapes).

When several of these are set on one reference they apply in a fixed order: `onEmpty` → `format` → `left`/`right` (padding) → `encode` → **`case` last**. Each step feeds the next, so `left`/`right` justify the **raw** value and `encode` escapes the padded result, while `case` runs last (casing the padding characters, the `onEmpty` default, and any encoded entities). Examples: `<ref:text out="ab" case="upper" left="4" padding="x"/>` → `"ABXX"`; `encode="xml" case="upper"` on `"<x>"` → `"&LT;X&GT;"`; `out="a<b" encode="xml" left="10" padding="-"` justifies the 3-char `a<b` to width 10 first, then escapes → `"a&lt;b-------"`.

## Examples

**Emit a literal string:**
```xml
<ref:text out="hi"/>
```
Produces: `"hi"`

**Compose a sentence from references.** Children of a `text` are concatenated in order:
```xml
<ref:text>
    <ref:text out="Hello, "/>
    <ref:user out="name"/>
    <ref:text out="!"/>
</ref:text>
```
Produces: `"Hello, Test User!"` (default context)

**Two-branch output (the `if/else` of references).** There is no `ref:if`; gate two `text` elements on opposite predicates so exactly one emits:
```xml
<ref:record file="master" out="filesize" store="@sz"/>
<ref:compare operator="gt" value1="@sz" value2="10000000" store="@big"/>
<ref:text onVariable="IsNotZero(@big)" out="LARGE"/>
<ref:text onVariable="IsZero(@big)"   out="SMALL"/>
```
Produces: `"LARGE"` (context: `{"records":[{"files":{"master":{"filesize":12000000}}}]}`). With a 5 MB file it produces `"SMALL"`.

**Gate on a field being populated:**
```xml
<ref:record out="status" store="@status"/>
<ref:text onVariable="IsNotEmpty(@status)" out="@status"/>
```
Produces: `"Approved"` (context: `{"records":[{"status":"Approved"}]}`). With an empty status, it emits nothing.

**Capture without emitting** (the basis of all silent logic):
```xml
<ref:text out="captured" store="@val"/>
```
Produces: `""` — the `store` swallows the output. Read `@val` later with an ungated `<ref:text out="@val"/>`.

## Gotchas

- **Only an ungated `text` emits.** A `ref:text` with `store="@val"` prints nothing; a gated `ref:text` prints only when its gate passes. Build all your logic with stores and gates, then finish with one plain `<ref:text out="...">`.
- **`onAllVariables`/`onAnyVariable` check whether each variable is DEFINED (non-null), not whether it is non-empty or true.** A variable stored as `""` still counts as defined and **passes**; only a variable that was *never stored* is false (and it does **not** error — it is simply treated as absent). A `compare` that evaluates false stores `"False"` (defined) and passes too. So `onAllVariables` emits when **every** listed variable has been set, `onAnyVariable` when **at least one** has. Use these for "was this variable ever assigned"; use `IsNotEmpty` for "is this field populated" and `IsNotZero`/`IsZero` for "is this condition true". (`onVariable` takes a predicate like `IsNotZero(@x)`; `onAnyVariable`/`onAllVariables` take bare `@variable` names, semicolon-separated.)
- **Close it with `</ref:text>`.** A mismatched `</text>` is a parse error.
- **Loose text outside the tags is literal output** — a reference is a template, so `id-<ref:eval expression="@n"/>` emits `id-` then the value. You don't need to wrap literals in `ref:text`. (Note: `@variables` interpolate only in attribute values, not in loose text.)

## See also
- [`object.md`](object.md) — the same grouping/gating, but preserves object types instead of coercing to text.
- [`../reference/patterns.md`](../reference/patterns.md) — store-then-emit, conditional defaults, two-branch output.
- [`../reference/operators-and-logic.md`](../reference/operators-and-logic.md) — predicates, gating, and combining conditions.
