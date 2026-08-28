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
- A **missing** field is not a collection at all, so `ref:count` on it THROWS
  rather than returning `0`. So does a field holding a plain string instead of a
  list. Guarding this correctly takes three parts, and the two obvious one-part
  guards both fail:

  - `onVariable="IsNotEmpty(@var)"` covers missing/empty only. A field whose
    value is a plain string passes the gate and then throws
    (`Attribute 'in' has an invalid value 'justonestring' for node 'count'.`),
    aborting the **whole reference**, not just the count.
  - `<ref:catch ex="System.Exception" out="0">` around a `ref:count` that carries
    `store=` does **not** put `0` into the variable. A `store` on an element that
    threw never happened, so the later read fails with
    `Variable '@cn' does not exist.`

  The form that actually works: pre-initialize, then count inside a catch.

  ```xml
  <ref:text out="0" store="@cn"/>
  <ref:catch ex="System.Exception" out=""><ref:count in="@list" store="@cn"/></ref:catch>
  <ref:text out="@cn"/>
  ```

  The pre-initialization is what survives when the count throws; the catch stops
  the throw from aborting the document.

## See also
- `record.md` / `field.md` — capturing field collections into a variable.
- `foreach.md` — iterate over the same collections.
- `compare.md` — branch on the resulting count.
- `../reference/patterns.md` — store-then-read patterns.
