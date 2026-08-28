# Reference idiom index (by programming intent)

Look up a programming construct → get the reference idiom. This is an **index**, not a manual —
it does not re-explain rules covered elsewhere:

- *Rules* of comparison, gating, AND/OR, and variable flow → `operators-and-logic.md`.
- *Every tag* and what it does → `catalog.md`.
- *Worked multi-step recipes* → `patterns.md`.
- *Execution model* (runs in document order; `store` assigns **and** suppresses output; only an
  ungated `ref:text` prints; define a variable before reading it) → `overview.md`.

## Intent → idiom

| Intent | Reference idiom | Example |
|---|---|---|
| assign / declare | `store=` | `<ref:text out="5" store="@num"/>` |
| read a variable | `@name` | `<ref:text out="@num"/>` |
| arithmetic | `ref:eval` (`+ - * / ( )` only) | `<ref:eval expression="(@av+@bv)*2" store="@res"/>` |
| compare → `True`/`False` | `ref:compare` | `<ref:compare value1="@num" value2="10" operator="gt" store="@big"/>` |
| boolean AND | nest `IsNotZero` gates | `<ref:object onVariable="IsNotZero(@aa)"><ref:text onVariable="IsNotZero(@bb)" out="X"/></ref:object>` |
| boolean OR | gated emit + nested `IsZero` fallback | `<ref:text onVariable="IsNotZero(@aa)" out="X"/><ref:object onVariable="IsZero(@aa)"><ref:text onVariable="IsNotZero(@bb)" out="X"/></ref:object>` |
| boolean NOT | gate on `IsZero` | `<ref:text onVariable="IsZero(@big)" out="X"/>` |
| `if` | gate output on a predicate | `<ref:text onVariable="IsNotZero(@big)" out="yes"/>` |
| `if / else` | two refs on opposite gates | `<ref:text onVariable="IsNotZero(@big)" out="yes"/><ref:text onVariable="IsZero(@big)" out="no"/>` |
| nested `if` (AND of flow) | gate inside a gated `ref:object` | `<ref:object onVariable="IsNotZero(@aa)"><ref:text onVariable="IsNotZero(@bb)" out="both"/></ref:object>` |
| `switch` / `case` | `ref:switch` + `item` + `default` | `<ref:switch keys="@type"><item key="A"><ref:text out="1"/></item><default><ref:text out="0"/></default></ref:switch>` |
| `for-each` (bounded) | `ref:foreach` over a collection | `<ref:foreach in="@list" storeitem="@item" join=", "><ref:text out="@item"/></ref:foreach>` |
| `try / catch` | `ref:catch` with fallback `out` | `<ref:catch ex="System.Exception" out="fallback"><!-- logic --></ref:catch>` |
| string replace | `ref:replace` | `<ref:replace in="@raw" oldValue=" " newValue="" store="@clean"/>` |
| regex extract (match only) | `ref:regex` (`match` is 1-based; no substitution) | `<ref:regex in="@text" expression="[0-9]+" match="1" store="@digits"/>` |
| format / pad / case | `format=`, `left=`/`right=`/`padding=`, `case=` | `<ref:text out="@num" left="4" padding="0"/>` |
| date math | `ref:datetimeeval` | `<ref:now store="@now"/><ref:datetimeeval in="@now" addDays="30" store="@due"/>` |
| date difference | `ref:datediff` | `<ref:datediff date1="@start" date2="@end" timeunit="days" store="@span"/>` |
| sequential counter | `ref:increment` | `<ref:increment counter="invoice" initialValue="1000" store="@id"/>` |
| macro / re-resolve text as a reference | `result="ref"` | `<ref:record fieldName="Template" result="ref"/>` |
| read input (field/attr/file) | `ref:record` | `<ref:record fieldName="SKU" out="value" store="@sku"/>` |
| call out / side effect | `ref:httpRequest`, `ref:increment` | see `httprequest.md` |

## Data types

string (default) · number (coerced inside `eval`/`compare`/`format`) · boolean = `True`/`False`
(from `compare`; gate with `IsNotZero`/`IsZero`) · collection (multi-value field / list / search result — iterate with `foreach`,
size with `count`) · datetime (`now`/`datetime`, math via `datetimeeval`, diff via `datediff`) ·
object (`ref:object` preserves type; `text` coerces to string). Empty-vs-missing-field semantics:
see `operators-and-logic.md`.

## No idiom exists for

- **`while` / unbounded loops** — `foreach`'s count is fixed by its input collection and there is
  no range generator; to loop N times, supply an N-element field. (No collection and need bounded
  iteration anyway? There's an advanced escape hatch — geometric-unroll + `result="ref"` — in
  `dynamic-references.md`.)
- **general recursion** — only `result="ref"` re-resolves, and it is **capped at 3 levels**.

There is NO block scope: a `store` inside a `foreach` body, a `switch` item, a `ref:object` or a `ref:catch` IS readable afterwards. The one exception is the `foreach` `storeitem`, which is consumed at loop end - reading it after the loop throws `Variable '@x' does not exist.`
don't export past a `foreach`/`switch`/`object`/`catch` — are detailed in `operators-and-logic.md`.)
