# ForEach reference (`ref:foreach`)

Iterates over the items of a collection, running its body once per item. With `join` it concatenates each item's output into a single delimited string; without it, the result is a collection of values. This is how you turn a multi-value field (Classification List, User List, Record List, …) into a formatted string. It is a **container tag**.

## Syntax
```xml
<ref:foreach in="@collection" storeitem="@item" [join=", "]>
    ...body, referencing @item...
</ref:foreach>
```

## Attributes
| Attribute | Description |
|---|---|
| `in` | The collection to iterate — usually a list-field value stored into a variable first. |
| `storeitem` | Variable holding the current item each pass, so the body can read its properties. Name is case-insensitive (`storeItem` works). |
| `join` | (optional) Separator for joining each item's output into one string. Without it, the output is a collection. |

## Examples

**Comma-separate the values of a list field.** Read the field, store it, loop with a `join`:
```xml
<ref:record fieldName="Brands" out="value" store="@brands"/>
<ref:foreach in="@brands" storeitem="@item" join=", ">
    <ref:text out="@item"/>
</ref:foreach>
```
Produces: `"Acme, Globex, Initech"` (context: `{"records":[{"fields":{"Brands":["Acme","Globex","Initech"]}}]}`)

**Without `join` you get a COLLECTION, and emitting it prints a .NET type name** - not the
items. This is real Aprimo behaviour, and a common way to ship a broken filename or label:
```xml
<ref:record fieldName="Brands" out="value" store="@brands"/>
<ref:foreach in="@brands" storeitem="@item">
    <ref:text out="@item"/>
</ref:foreach>
```
Produces: `"System.Collections.Generic.List`1[System.Object]"` - **always supply `join`** when the loop output is emitted.

To render a list *without* a loop, emit the variable directly: `<ref:text out="@brands"/>`
coerces it to `"Acme, Globex"` - the separator is a comma **and a space**.

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

- **Attribute names are case-insensitive.** `storeitem`, `storeItem` and `STOREITEM` all work. The tag-specific attributes are `in` and `storeitem`.
- **`join` vs. no `join`:** with `join`, items are concatenated using the separator; without it, the body produces a collection (which a text context will then comma-join on its own).
- **`in` must be defined earlier** with `store=`; an undefined variable is a used-before-defined error.
- **The `storeitem` itself is CONSUMED at loop end.** Reading it after the loop throws `Variable '@item' does not exist.` - even when the loop ran zero times. Only body `store=` variables survive.
- **A `store` inside the loop body DOES survive the loop.** There is no block scope in references (see [`../reference/patterns.md`](../reference/patterns.md)) - the variable holds whatever the last iteration that executed the element assigned. This is what makes the sentinel "first item" idiom work, and it means you do not need `ref:increment` just to carry a value out of a loop. The one thing that does not survive is an element that never ran: if a gate suppressed it on every iteration, its variable was never created and reading it throws.

## See also
- [`record.md`](record.md) — reading the list field you iterate, and per-item file/version properties.
- [`switch.md`](switch.md) — branching on each item's value inside the loop body.
- [`../reference/patterns.md`](../reference/patterns.md) — building delimited strings and joining collections.

## Switching on the loop item

`switch keys="@item"` directly on a `storeitem` does **not** work - a loop item
is a typed value and `keys=` needs text. Re-store the item through `ref:text`
first, then switch on the copy:

```xml
<ref:foreach in="@tags" storeitem="@tag" join=", ">
  <ref:text out="@tag" store="@tagKey"/>
  <ref:switch keys="@tagKey">
    <item key="alpha">A</item>
    <default>other</default>
  </ref:switch>
</ref:foreach>
```

The same applies to `ref:compare value1="@tag"` - compare the re-stored copy.
