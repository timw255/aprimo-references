# Operators, logic, and variable flow

This is where references most often break. Every rule here is enforced by the compiler — when in
doubt, write the smallest version and run it through `lint`/`execute` to confirm. (For a quick
"which ref does this programming construct" lookup, see `computational-model.md`; this file is the
detailed rules behind those idioms.)

## Comparison operators (closed set)

`ref:compare` takes `value1`, `value2`, `operator`, and `store`. The operator MUST be one of:

| Meaning | Accepted spellings (compiler-verified) |
|---|---|
| equal | `equal`, `eq`, `==` |
| not equal | `notEqual`, `ne`, `!=` |
| greater than | `greaterThan`, `gt` |
| greater or equal | `greaterThanOrEquals`, `ge` |
| less than | `lessThan`, `lt` |
| less or equal | `lessThanOrEquals`, `le` |

That is the entire set. **Verified rejections:** the only symbol forms are `==` and `!=` — `>`,
`>=`, `<`, `<=` are **not** accepted. It's `ge`/`le`, **not** `gte`/`lte`. And there is **no
`contains`, `startsWith`, `in`, `between`, `matches`, `notContains`.** For substring matching use
`ref:regex`; for "is X one of a list", do one `compare` per candidate and OR the results (see
below) or use a `switch`. `compare` resolves to the strings **`True`** (true) or **`False`** (false)
into its `store` variable — **not** `1`/`0`. The result is still a boolean usable by `IsNotZero`
(true for `True`) / `IsZero` (true for `False`) gates.

**How ordering comparisons (`gt`/`ge`/`lt`/`le`) decide their type (verified):**

- If **both** values are numbers, the comparison is **numeric** — e.g. `gt("2","10")` is `False`
  (2 is not greater than 10 numerically).
- Else if **both** values parse as dates, the comparison is **datetime**.
- **Otherwise it falls back to ordinal/lexicographic string comparison.** Text *does* order —
  there is no "ordering only works on numbers" limitation; just know mixed/text comparisons are
  ordinal. Verified: `gt("Banana","Apple")` is `True`, `lt("Apple","Banana")` is `True`, and
  `gt("abc","5")` is `True` (text, because `"abc"` isn't a number).

`eq`/`ne` compare as **strings** (with date-equality when both parse as dates).

```xml
<ref:record fieldName="FileSize" out="value" store="@size"/>
<ref:compare operator="gt" value1="@size" value2="10000000" store="@big"/>
<ref:text onVariable="IsNotZero(@big)" out="LARGE"/>
<ref:text onVariable="IsZero(@big)"   out="SMALL"/>
```

## Gating (conditional output)

There is no `ref:if`. You make output conditional by putting a **gate attribute** on a `ref:text`
or `ref:object`:

- `onVariable="<predicate>"` — emit only if the predicate is true. Predicates:
  - `IsNotEmpty(@var)` / `IsEmpty(@var)` — is the variable set / unset
  - `IsNotZero(@var)` / `IsZero(@var)` — is a stored compare result true / false. `IsNotZero` is
    true for `True` or a nonzero **integer**; `IsZero` is true for `False` or the integer `0`. The
    number must be a whole integer — a decimal like `"0.0"` or `"0.5"` is **neither** zero nor
    non-zero — and an **empty** value is also **neither**.
- `onAllVariables="@first; @second"` — AND-style gate: fires only if **every** listed variable is
  **defined** (has been `store`d — a stored `""` still counts), not whether it's non-empty or true
- `onAnyVariable="@first; @second"` — OR-style: fires if **at least one** is defined

  (Names are semicolon-separated; a comma is rejected. A *never-stored* variable is the only thing
  these treat as absent — and it does not error.)

**Critical gotcha:** a `compare` always *stores* `"True"` or `"False"` — both **defined** — so
`onAllVariables`/`onAnyVariable` over compare results fire whenever the compares *ran*, regardless of
whether they were true. Use these for "was this variable ever set"; for "are these conditions both
*true*", use `IsNotZero` gates (below). Confirm with `execute` on inputs that should fail.

## Combining conditions

Combine conditions with **gates**, not arithmetic. **Do NOT use `eval`:** `ref:eval` is
**numeric-only**, and feeding it a `compare` result (`True`/`False`) **errors** ("invalid number").
The old "AND = eval product, OR = eval sum" idiom does **not** work on real Aprimo.

**AND (both true)** — nest `IsNotZero` gates (verified):

```xml
<ref:record fieldName="Status" out="value" store="@status"/>
<ref:record file="master" out="filesize" store="@size"/>
<ref:compare operator="eq" value1="@status" value2="Approved" store="@isAppr"/>
<ref:compare operator="gt" value1="@size" value2="1000" store="@isBig"/>
<ref:object onVariable="IsNotZero(@isAppr)">
  <ref:text onVariable="IsNotZero(@isBig)" out="PUBLISH"/>
</ref:object>
```

**OR (either true)** without double-emit — emit on the first, then gate the second on the first
being false (verified):

```xml
<ref:text onVariable="IsNotZero(@isAppr)" out="HIT"/>
<ref:object onVariable="IsZero(@isAppr)">
  <ref:text onVariable="IsNotZero(@isBig)" out="HIT"/>
</ref:object>
```

Two gotchas these rely on (both verified): `store=` **suppresses emission** (storing refs print
nothing — only an ungated `text` emits), and an **empty value fails a compare gracefully**
(`compare gt value1="" ...` → `False`, no crash), so you can run every `compare` unconditionally
and they'll all be defined before their gates.

## Branching: switch

For multi-way mapping, `ref:switch` is cleaner than stacked compares. Children are `<item
key="...">` (exact-match) and an optional `<default>`. Keys match **exactly** — there are no range
or regex keys (`<item key="[0-5]">` does NOT match numbers 0–5; it matches the literal string
`[0-5]`). For ranges, use `compare` + gates instead.

```xml
<ref:record fieldName="AssetType" out="value" store="@type"/>
<ref:switch keys="@type">
  <item key="Design"><ref:text out="5 years"/></item>
  <item key="Video"><ref:text out="5 years"/></item>
  <default><ref:text out="3 years"/></default>
</ref:switch>
```

## Variable flow (read before use)

- **Define before use.** A variable must be populated by a `store="@val"` *earlier in document order*
  than any reference that reads `@val`. Reading an undefined variable throws at run time
  (`Variable '@val' does not exist.`) — a catchable error — and lint warns about it.
- **There is no block scope.** A variable `store`d inside a `switch` item, a `foreach` body, a
  `ref:object` or a `ref:catch` **is** readable by later siblings and after the block. You do not
  need to hoist it to the top level.
- **But a `store` that never RUNS leaves the variable undefined.** A gate that is false, or a
  block that throws before reaching the `store`, performs no assignment — the variable keeps its
  previous value, or stays undefined if it never had one. That is what makes seeding matter:
  `<ref:text out="" store="@flag"/>` before a conditional block, so a later
  `onVariable="IsNotEmpty(@flag)"` has something to read. See *Gating* in `../docs/overview.md`.
- Variable names are `@` + a letter followed by one or more letters/digits — a name must be **at
  least two characters**, so a single letter like `@s` is **not** a valid name. An **underscore ends
  the name** (`@base_Prev` reads the variable `@base` followed by the literal text `_Prev`, and
  `@base_x` reads `@base` then `_x`), so don't put `_` inside a variable you mean to read back.
- **A missing field reads as NULL**, which renders empty but is *not* the same as an empty-but-present
  field. Reading it does not throw, but passing the null to `ref:switch`, `ref:replace`, `ref:regex`,
  `ref:count` or `ref:datediff` does — see [Missing values are null](../docs/overview.md#missing-values-are-null).
  (Unknown-entity *lookups* — `ref:user`/`ref:usergroup`/`ref:classification` by a bad name/id — also throw.)

## Lint-clean is not enough — execute

Some things pass `lint` but misbehave at **runtime**, so always `execute` to confirm:

- An `out` value with wrong casing (`out="Id"`, `out="Culture"`) lints fine but returns **empty** —
  `out` values must be **lowercase** at runtime.
- `collection out="value"` lints valid but **crashes** at execution.
- `regex` accepts a `replace` attribute but **ignores** it (it only ever returns a match) — use
  `ref:replace` for substitution.

## Date math

Never write `now+5` as text. Compute dates with `ref:datetimeeval`:

```xml
<ref:now store="@now"/>
<ref:datetimeeval in="@now" addYears="5" store="@future"/>
<ref:text out="@future"/>
```

`addYears`, `addMonths`, `addDays`, `addHours`, `addMinutes`, `addSeconds` — combine them, and use
negatives to subtract (`addDays="-30"`). For "days between two dates" use `ref:datediff`
(`date1`, `date2`, `timeunit`, `store`). Read `docs/datetimeeval.md` and
`docs/datediff.md` for specifics, and confirm with `execute`.

## When unsure — the move is always the same

Write the smallest reference that expresses the piece you're unsure about, run it through `execute`
with a context that should make it true and one that should make it false, and look at the two
outputs. The compiler will tell you the truth faster than reasoning about it will.
