# Count reference (`ref:count`)

Returns the number of items in a collection that was captured into a variable earlier in the reference.

## Syntax
```xml
<ref:count in="@collection"/>
```

## Attributes
| Attribute | Required | Description |
|---|---|---|
| `in` | yes | A variable holding a collection — e.g. a multi-value field, a Classification List field, or a list-valued `out` such as `groupnames`. The variable must be defined earlier by a `store`. |

## Examples

Count a user's group memberships:
```xml
<ref:user out="groupnames" store="@grps"/><ref:count in="@grps"/>
```
Produces: `"2"`

Count a multi-value record field (context: `Brands` = `["Nike","Adidas","Puma","Reebok"]`):
```xml
<ref:record fieldName="Brands" store="@brands"/><ref:count in="@brands"/>
```
Produces: `"4"`

An empty collection counts as zero (context: `Brands` = `[]`):
```xml
<ref:record fieldName="Brands" store="@brands"/><ref:count in="@brands"/>
```
Produces: `"0"`

Capture the count itself for later use, then emit it:
```xml
<ref:user out="groupnames" store="@grps"/><ref:count in="@grps" store="@num"/><ref:text out="@num"/>
```
Produces: `"2"`

## Gotchas
- **The collection variable must be defined before `ref:count` reads it.** If `in` points at a variable that was never `store`d (or a global), it throws at run time (`Variable '@nope' does not exist.`); lint warns about it.
- The source/UI doc spells the record lookup `fieldname`, but the compiler requires the camelCase `fieldName`.
- `count` emits a number as text. To branch on it, capture with `store` and feed the variable into a `ref:compare` (`value1`/`operator`/`value2`).
- An empty collection produces `0`, not an empty string.

## See also
- `record.md` / `field.md` — capturing field collections into a variable.
- `foreach.md` — iterate over the same collections.
- `compare.md` — branch on the resulting count.
- `../reference/patterns.md` — store-then-read patterns.
