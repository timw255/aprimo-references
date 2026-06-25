# Proven patterns (cookbook)

Idioms verified against the compiler. Copy, adapt, and re-verify with `execute` for your case.
These solve the things that trip up reference authors most.

## How references map to code

References compose like a small program, so structure them with your usual judgment — the building
blocks are:

| Reference | Acts like |
|---|---|
| `store="@v"` / `@v` | assignment / read |
| `onVariable` / `onAllVariables` / `onAnyVariable` | `if` (see [operators-and-logic](operators-and-logic.md)) |
| `ref:switch` | `switch` |
| `ref:catch` | `try` / `catch` |
| `ref:foreach` | a loop over a collection |
| `ref:object`, or `ref:text` with children | a block / scope |

(For a fuller intent→idiom lookup — boolean AND/OR/NOT, loops, `try/catch`, what has *no* idiom — see
[computational-model.md](computational-model.md). When a goal needs state or periodic checks *beyond*
a single reference — extra fields, a Rule, a nightly schedule — see
[automation-patterns.md](automation-patterns.md).)

What's worth knowing are the few Aprimo-specific behaviors that aren't obvious from the syntax — they
decide when grouping actually buys you something:

- **An attribute on a container applies to its whole assembled body.** `onVariable`/`onAllVariables`/
  `onAnyVariable`, `case`, `encode`, `format`, `left`/`right`, and `store` on a `ref:object` (or a
  `ref:text` with children) act on the concatenation of everything inside — so one gate or one
  transform can cover a group rather than being repeated per child. (Verified: `<ref:object
  onVariable="IsNotZero(@ok)">A<.../>C</ref:object>` emits the whole block or nothing;
  `<ref:object case="upper">…</ref:object>` uppercases all of it.)
- **A gated `store` only assigns when its gate passes** — a false gate skips it, leaving the variable
  unchanged (it does **not** store `""`). That makes conditional assignment and an "if/else that yields
  a value" behave as expected (see *Pick a value by condition* below).
- **Loose text between tags is literal output and `<ref>` tags interpolate**, so a sentence is a
  template, not a stack of `ref:text` fragments (`Total: <ref:eval expression="@a*@b"/> items`).
  `@variables` interpolate in **attribute values**, not in loose text.
- **A container body is its own scope** — a `store` inside a `switch` item or `foreach` body may not be
  visible afterward; lift it to the outer level if you need it later.

The rest is the normal call you'd make in any language about when a block clarifies and when it's just
noise.

## `store` suppresses emission

A reference with `store="@val"` captures its value into the variable **without printing it** to the
output — even with `out="value"`. Only an *ungated* `ref:text` (or a tag with no `store`) emits.
This is what lets you read a dozen fields and run a dozen compares, yet output only the final
result. Verified:

```
<ref:record fieldName="X" out="value" store="@val"/>  <!-- reads & stores, emits nothing -->
<ref:text out="done"/>                                <!-- output is just: "done" -->
```

So: **storing refs are silent; the only output is what your ungated `text` emits.**

## Set up at the top, emit at the bottom

Do every field read and any variable setup in a block at the **top**, and put your output text
**last**. This one habit avoids the two things that bite people:

- **Undefined-variable errors.** Reading a variable that was never `store`d throws at run time.
  Doing all the stores first guarantees each variable exists before it's read. (You usually don't
  even need to initialize: a field read of a missing or empty field stores `""`, so a field-backed
  variable is always safe to read — `<ref:record fieldName="Maybe" out="value" store="@m"/>` leaves
  `@m=""` if `Maybe` doesn't exist, never undefined.)
- **Stray whitespace.** A line that emits nothing — an init, or a storing field read — placed
  **between** output lines leaks the surrounding whitespace into the result. The newline *and the
  indentation* on each side become spaces, with nothing emitted between them:

  ```xml
  START
      <ref:text out="" store="@iv"/>
      END
  ```
  → `"START          END"` (verified against Aprimo — a 10-space gap). The **same line at the top**
  is harmless: leading whitespace is trimmed off the final result, so `"HELLO"` stays `"HELLO"`.

So: **read/compute first (silently), emit last.** Initialize only the variables that are set
*conditionally* (inside a gate, `switch` item, or `foreach` body, where the `store` might not run) —
not every variable — and keep those inits in the top block with the field reads:

```xml
<ref:record fieldName="Status" out="value" store="@status"/>   <!-- field reads: silent, safe -->
<ref:compare value1="@status" value2="Approved" operator="eq" store="@isHit"/>
<ref:text out="" store="@picked"/>                             <!-- only the conditionally-set var -->
<ref:text onVariable="IsNotZero(@isHit)" out="A" store="@picked"/>
Result: <ref:text out="@picked"/>                              <!-- output begins here -->
```

Prefer this structure over **minifying** the whole reference onto one line: minifying also removes
the whitespace, but it costs readability — the top/bottom split keeps the reference legible. If you
*can't* avoid a silent line mid-output, wrap the read in `ref:catch out=""` instead of initializing.

> One nuance: a variable that took its value from a **missing** field reads as `""` and is safe, but
> a defined-gate (`onVariable`/`onAllVariables`) treats that as **not set**. An explicit
> `<ref:text out="" store="@v"/>` makes it count as **set**. So initialize explicitly when you need a
> defaulted variable to *pass* a gate.

## Combining conditions with gates

`ref:compare` stores the string **`True`** or **`False`** (not `1`/`0`). You **cannot** feed those
into `ref:eval` to AND/OR them — eval is numeric-only and a `True`/`False` operand throws "an
invalid number was specified". Combine conditions with `IsNotZero`/`IsZero` gates instead.
`IsNotZero(@cond)` is true when `@cond` is `True` (or a nonzero number); `IsZero(@cond)` is true
when `@cond` is `False` (or numeric `0`).

**AND (both true)** — nest `IsNotZero` gates:

```xml
<ref:compare value1="@status" value2="Approved" operator="eq" store="@isAppr"/>
<ref:compare value1="@size" value2="1000" operator="gt" store="@isBig"/>
<ref:object onVariable="IsNotZero(@isAppr)">
  <ref:text onVariable="IsNotZero(@isBig)" out="PUBLISH"/>
</ref:object>
```

**OR (either true)** — emit on the first; in the else-branch (`IsZero` of the first) test the
second, so you never double-emit when both are true:

```xml
<ref:text onVariable="IsNotZero(@isAppr)" out="HIT"/>
<ref:object onVariable="IsZero(@isAppr)">
  <ref:text onVariable="IsNotZero(@isBig)" out="HIT"/>
</ref:object>
```

## Pick a value by condition (an "if/else that yields a value")

A **gated `store` only assigns when its gate passes** — a false gate *skips* the assignment and leaves
the variable at its prior value (it does **not** store `""`). So two opposite gated stores to the same
variable cleanly pick one value — the natural `if/else`:

```xml
<ref:text onVariable="IsNotZero(@cond)" out="A" store="@pick"/>  <!-- sets @pick="A" only if cond -->
<ref:text onVariable="IsZero(@cond)"   out="B" store="@pick"/>   <!-- else sets @pick="B" -->
<!-- @pick is "A" or "B" -->
```

Equivalently, **wrap the gated branches in a parent that carries the `store`** — exactly one branch
fires, so the parent's output *is* the chosen value. Use **`ref:object`** instead of `ref:text` when
the branches are non-string values you want to keep typed (`text` coerces to string):

```xml
<ref:compare value1="@first" value2="@second" operator="gt" store="@firstWins"/>
<ref:object store="@max"><ref:text onVariable="IsNotZero(@firstWins)" out="@first"/><ref:text onVariable="IsZero(@firstWins)" out="@second"/></ref:object>
```

> **Keep the wrapped branches on one line.** Whitespace *between* the child elements is preserved as
> output (each newline/tab becomes a single space — see [overview.md](../docs/overview.md#whitespace)),
> so if you split the branches across indented lines, the stored value picks up those spaces (`" A B "`).

## Run every `compare` unconditionally

So that every condition variable is defined before the gates that read it, run each `compare` at the
top level (not inside a gate). `compare` always writes `True`/`False` to its `store`. An *empty*
input fails gracefully — `compare operator="gt" value1="" value2="2026-12-31"` returns `False`, it
does **not** crash — so you usually don't need to special-case empty.

## Trim before a numeric compare

A value with hidden whitespace (` 10485760 `) silently fails a numeric compare — `ge` returns `0`.
Strip it first with `ref:replace`:

```xml
<ref:record file="master" out="filesize" store="@raw"/>
<ref:replace in="@raw" oldValue=" " newValue="" store="@size"/>
<ref:compare operator="ge" value1="@size" value2="10485760" store="@ok"/>
```

## Crash-safety: wrap in `catch`

For a rule that must never error (validation rules, default values), wrap everything in `ref:catch`
with the fallback as `out`. Any exception inside resolves to that value:

```xml
<ref:catch ex="System.Exception" out="false">
  ... all logic; final text emits "true"/"false" ...
</ref:catch>
```

## Conditional default (only set when X and currently empty)

Read the field, gate on the condition(s), and only the matching branch emits — everything else
emits nothing (so the field keeps its existing value):

```xml
<ref:record fieldName="Status" out="value" store="@status"/>
<ref:record fieldName="Target" out="value" store="@existing"/>
<ref:compare operator="eq" value1="@status" value2="Retained" store="@isRetained"/>
<ref:object onVariable="IsNotZero(@isRetained)">
  <ref:object onVariable="IsEmpty(@existing)">
    <ref:now store="@now"/>
    <ref:datetimeeval in="@now" addYears="5" store="@dt"/>
    <ref:text out="@dt"/>
  </ref:object>
</ref:object>
```

Nesting two gates is the reliable way to AND conditions for control flow (see "Combining conditions
with gates" above).

## Date before/after

`compare` works directly on date strings — no `datediff` needed for simple before/after:

```xml
<ref:compare operator="gt" value1="@expiration" value2="2026-12-31" store="@future"/>
```
For "how many days/years between", use `ref:datediff` (`date1`, `date2`, `timeunit`, `store`).

## Multi-way mapping with a fallthrough default

`switch` keys are exact-match. Map the exceptions explicitly and let `default` handle the rest:

```xml
<ref:switch keys="@subtype">
  <item key="Sell Sheets"><ref:datetimeeval in="@now" addYears="10" store="@dt"/><ref:text out="@dt"/></item>
  <default><ref:datetimeeval in="@now" addYears="5" store="@dt"/><ref:text out="@dt"/></default>
</ref:switch>
```
