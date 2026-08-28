# SearchExpression (`ref:searchExpression`)

Builds a search expression by substituting parameter values into `?` placeholders. Commonly used to filter the options shown in a list field (User List, Option List, etc.) based on other field values.

## Syntax
```xml
<ref:searchExpression expression="expression with ? placeholders" param1="value" param2="value" ... paramN="value"/>
```

The emitted result is the `expression` string with each `?` replaced, in order, by `param1`, `param2`, … `paramN`.

## Attributes
| Attribute | Required | Description |
|---|---|---|
| `expression` | Yes | The search-expression template. Each `?` is a positional placeholder filled from the parameters. |
| `param1`, `param2`, … `paramN` | No | Parameter values substituted into the `?` placeholders, left to right. Must start at `param1` and be numbered consecutively. A parameter value may be a literal or an `@variable` defined by a preceding `store`. |

## Examples

> **The engine NORMALIZES what it emits.** It wraps the whole expression in
> parentheses, spaces out the operators, lowercases `AND`/`OR`, and **adds the
> quotes around every substituted value itself**. The output is not the text you
> typed - if another system consumes the string, expect the normalized form.

Limit a User List field to users whose name begins with `a`:
```xml
<ref:searchExpression expression="name=?" param1="a*"/>
```
Produces: `(name = 'a*')`

Multiple placeholders are filled in order:
```xml
<ref:searchExpression expression="name=? and status=?" param1="a*" param2="active"/>
```
Produces: `(name = 'a*' and status = 'active')`

Use a value from another field - the **store-then-use** pattern:
```xml
<ref:record fieldName="Region" store="@region"/>
<ref:searchExpression expression="group.name=?" param1="@region"/>
```
Produces: `(group.name = 'EMEA')`

**Never quote the `?`.** A `?` inside quotes is a literal, not a placeholder -
the parameter is silently dropped with **no error**, and you ship a broken
search string:
```xml
<ref:searchExpression expression="name = '?'" param1="@region"/>
```
Produces: `(name = '?')` - the value never appears. Write `expression="name = ?"`.

**Literal values use SINGLE quotes only.** Double quotes are rejected
(`The search expression contains an invalid value.`):
```xml
<ref:searchExpression expression="name = 'my name'"/>
```
Produces: `(name = 'my name')`

**Values are escaped for you.** A `'` inside a parameter is doubled
automatically - `O'Brien` becomes `'O''Brien'`. Do **not** pre-escape it, or you
will get `'O''''Brien'`.
## Gotchas
- **Parameters are positional.** The Nth `?` is replaced by `paramN`. Provide as many parameters as there are placeholders.
- **Parameter names must start at `param1` and increment with no gaps** (`param1`, `param2`, …). A maximum of 25 parameters is supported.
- **A `@variable` used in a parameter must be defined by a preceding `store`** (e.g. a `ref:record ... store="@region"` line). Otherwise it throws at run time (`Variable '@region' does not exist.`); lint warns about it.

## See also
- `record.md` — reading the field value used as a parameter.
- `../reference/patterns.md` — store-then-use and list-filtering patterns.
