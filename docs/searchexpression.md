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

Limit a User List field to users whose name begins with `a`:
```xml
<ref:searchExpression expression="name=?" param1="a*"/>
```
Produces: `name=a*`

Multiple placeholders are filled in order:
```xml
<ref:searchExpression expression="name=? and status=?" param1="a*" param2="active"/>
```
Produces: `name=a* and status=active`

Use a value from another field — the **store-then-use** pattern. Here the `Region` field holds `EMEA`:
```xml
<ref:record fieldName="Region" store="@region"/>
<ref:searchExpression expression="group.name=?" param1="@region"/>
```
Produces: `group.name=EMEA`

Quotes inside the expression — escape with `&quot;`:
```xml
<ref:searchExpression expression="name = &quot;my name&quot;"/>
```
Produces: `name = "my name"`

…or use single quotes:
```xml
<ref:searchExpression expression="name = 'my name'"/>
```
Produces: `name = 'my name'`

## Gotchas
- **Parameters are positional.** The Nth `?` is replaced by `paramN`. Provide as many parameters as there are placeholders.
- **Parameter names must start at `param1` and increment with no gaps** (`param1`, `param2`, …). A maximum of 25 parameters is supported.
- **Quotes:** since attribute values are XML-quoted, embed literal quotes as `&quot;` or use single quotes inside the expression.
- **A `@variable` used in a parameter must be defined by a preceding `store`** (e.g. a `ref:record ... store="@region"` line). Otherwise it throws at run time (`Variable '@region' does not exist.`); lint warns about it.

## See also
- `record.md` — reading the field value used as a parameter.
- `../reference/patterns.md` — store-then-use and list-filtering patterns.
