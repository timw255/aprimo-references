# ForEach reference (`ref:foreach`)

Iterates over the items of a collection, running its body once per item. With `join` it concatenates each item's output into a single delimited string; without it, the result is a collection of values. This is how you turn a multi-value field (Classification List, User List, Record List, …) into a formatted string. It is a **container tag**.

## Syntax
```xml
<ref:foreach in="@collection" storeitem="@item" [join=", "] [limit="N"] [orderBy="@item"]>
    ...body, referencing @item...
</ref:foreach>
```

## Attributes
| Attribute | Description |
|---|---|
| `in` | The collection to iterate — usually a list-field value stored into a variable first. |
| `storeitem` | Variable holding the current item each pass, so the body can read its properties. **Lowercase** — `storeItem` is rejected. |
| `join` | (optional) Separator for joining each item's output into one string. Without it, the output is a collection. |
| `limit` | (optional) Maximum number of items to process (applied by the Aprimo runtime). |
| `orderBy` | (optional) Sort the items before iterating (applied by the Aprimo runtime). |

## Examples

**Comma-separate the values of a list field.** Read the field, store it, loop with a `join`:
```xml
<ref:record fieldName="Brands" out="value" store="@brands"/>
<ref:foreach in="@brands" storeitem="@item" join=", ">
    <ref:text out="@item"/>
</ref:foreach>
```
Produces: `"Acme, Globex, Initech"` (context: `{"records":[{"fields":{"Brands":["Acme","Globex","Initech"]}}]}`)

**Without `join`, you get a collection** (here coerced to its default comma form by the text context):
```xml
<ref:record fieldName="Brands" out="value" store="@brands"/>
<ref:foreach in="@brands" storeitem="@item">
    <ref:text out="@item"/>
</ref:foreach>
```
Produces: `"Acme,Globex"` (context: `{"records":[{"fields":{"Brands":["Acme","Globex"]}}]}`)

**Read a per-item property inside the loop.** For collections of objects (user lists, classification lists), resolve each item with the matching reference. Here, the current user's groups joined with a custom separator:
```xml
<ref:user out="groupNames" store="@grps"/>
<ref:foreach in="@grps" storeitem="@item" join="; ">
    <ref:text out="@item"/>
</ref:foreach>
```
Produces: `"Designers; Marketing"` (default context)

**Resolve classification name paths** from a list of classification ids (one `classification` lookup per item):
```xml
<ref:record fieldName="Categories" out="value" store="@cats"/>
<ref:foreach in="@cats" storeitem="@item" join=", ">
    <ref:classification id="@item" out="namePath"/>
</ref:foreach>
```
(`out="namePath"` is camelCase — `namepath` is rejected.)

## Gotchas

- **`storeitem` is lowercase.** `storeItem` (camelCase) is a parse error: `ref:foreach has invalid attribute 'storeItem'`. The valid tag-specific attributes are exactly `in`, `storeitem`, `limit`, `orderBy`.
- **`join` vs. no `join`:** with `join`, items are concatenated using the separator; without it, the body produces a collection (which a text context will then comma-join on its own).
- **`limit`/`orderBy` are runtime concerns.** They lint clean and are accepted, but ordering and truncation are applied by the live Aprimo runtime — this transpiler does not reorder or truncate the output, so don't expect its preview to reflect them.
- **`in` must be defined earlier** with `store=`; an undefined variable is a used-before-defined error.
- **Variables stored inside the loop may not survive it.** If you need a running total or a value afterward, prefer `ref:increment`/top-level stores; treat the loop body as per-item scope.

## See also
- [`record.md`](record.md) — reading the list field you iterate, and per-item file/version properties.
- [`switch.md`](switch.md) — branching on each item's value inside the loop body.
- [`../reference/patterns.md`](../reference/patterns.md) — building delimited strings and joining collections.
