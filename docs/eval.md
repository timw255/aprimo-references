# Eval (`ref:eval`)

Evaluates a simple **numeric** arithmetic expression and returns the calculated value. Use it for real math on numeric values — field numbers, counters, dates-as-numbers — not for combining conditions.

> **Operands must be numeric.** `ref:eval` does `+ - * / ( )` on numbers only. A `ref:compare` result is the string **`True`** or **`False`** (not `1`/`0`), and feeding that — or any non-numeric value — into `ref:eval` **errors** ("an invalid number was specified"). The old "AND = product / OR = sum" idiom does **not** work on real Aprimo. To combine conditions, use `IsNotZero`/`IsZero` gates instead — see [../reference/operators-and-logic.md](../reference/operators-and-logic.md) and [../reference/patterns.md](../reference/patterns.md).

## Syntax
```xml
<ref:eval expression="(2+3)*5"/>
```

## Attributes
| Attribute | Description |
|---|---|
| expression | The arithmetic expression. May reference `@variables`, which are substituted before evaluation. |
| store | (optional) Variable to capture the result. With `store`, the eval emits nothing. |

### Allowed operators
Only `+`, `-`, `*`, `/`, and grouping with `(` `)`. There is **no `%` / modulo operator**; `10%3` fails with "An invalid number was specified in the expression." - a message that points at the operand, not at the operator, so it reads like a data problem when it is a syntax one. There are no comparison, logical, or boolean operators, and every operand must resolve to a **number**. To express logic, do not feed compare results into eval — combine conditions with `IsNotZero`/`IsZero` gates (see [../reference/patterns.md](../reference/patterns.md)).

## Examples

**Basic arithmetic** with grouping:
```xml
<ref:eval expression="(2+3)*5"/>
```
Produces: `"25"`

**Division can yield a decimal:**
```xml
<ref:eval expression="10 / 4"/>
```
Produces: `"2.5"`

**Increment a stored numeric value:**
```xml
<ref:record fieldName="ViewCount" out="value" store="@count"/>
<ref:eval expression="@count + 1"/>
```
If `@count` is `7`, produces: `"8"`

**Arithmetic across fields** with grouping:
```xml
<ref:eval expression="(@width + 10) * @scale"/>
```
Substitutes the numeric field values, then evaluates.

> **Do not** feed a `ref:compare` result into `ref:eval`. `<ref:eval expression="@isAppr * @isBig"/>` errors because `@isAppr`/`@isBig` are `True`/`False`, not numbers. To AND/OR conditions, use `IsNotZero`/`IsZero` gates — see [../reference/patterns.md](../reference/patterns.md).

## Gotchas
- **Operands must be numeric.** Every term must resolve to a number. A `ref:compare` result is `True`/`False` and will throw "an invalid number was specified" inside eval. Combine conditions with gates, not arithmetic.
- **`store` suppresses output.** An eval with `store="@val"` prints nothing; only an ungated `ref:text` emits.
- **Variables must be defined earlier in document order.** Reading an undefined variable throws at run time (`Variable '@x' does not exist.`) — a catchable error, so wrap risky logic in `ref:catch` when a variable might be missing.
### Division by zero does not throw

Arithmetic is double-precision, and dividing by zero produces a **value**, not an error:

| Expression | Result |
|---|---|
| `100/0` | `Infinity` |
| `0-100/0` | `-Infinity` |
| `0/0` | `NaN` |

This is verified live Aprimo behaviour, and it is the most dangerous thing on this page, because the obvious defence does not work:

```xml
<!-- WRONG: the catch never fires, and "Infinity" ships into your output -->
<ref:catch ex="System.Exception" out="n/a"><ref:eval expression="@total/@count"/></ref:catch>
```

Guard the **divisor** instead:

```xml
<ref:compare value1="@count" value2="0" operator="gt" store="@hasItems"/>
<ref:eval onVariable="IsNotZero(@hasItems)" expression="@total/@count" store="@avg"/>
<ref:text onVariable="IsZero(@hasItems)" out="n/a" store="@avg"/>
<ref:text out="@avg"/>
```

### Results are shortest-round-trip doubles

`ref:eval` emits the shortest string that round-trips the double, so a division can produce a long tail of digits: `1261/3` is `420.3333333333333` and `0.1+0.2` is `0.30000000000000004`. There is no precision or rounding attribute. If you are showing a number to a person, round it yourself (multiply, truncate with `left=`, or `ref:regex`) rather than emitting the raw quotient.

- **Stick to `+ - * / ( )`.** There are no comparison or boolean operators. Use `ref:compare` (which yields `True`/`False`) and `IsNotZero`/`IsZero` gates for logic.

See also: [../reference/operators-and-logic.md](../reference/operators-and-logic.md) and [../reference/patterns.md](../reference/patterns.md).
