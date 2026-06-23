# Collection (`ref:collection`)

Retrieves the Id or name of the current collection, or of a collection selected by Id, name, or search expression.

## Syntax
```xml
<ref:collection [id="id" | name="name" | search="expression"] out="out"/>
```

Omit `id`/`name`/`search` to return information about the **current** collection (the collection in scope when the reference runs, e.g. the collection a rule is triggered on).

## Attributes
| Attribute | Required | Description |
|---|---|---|
| `out` | Yes | The property to return. Valid values: `id`, `name`. (See note on `value` under Gotchas.) |
| `id` | No | Selects the collection by its Id (Guid). |
| `name` | No | Selects the collection by its name. |
| `search` | No | Selects the collection via a search expression (should resolve to exactly one collection). |
| `store` | No | Captures the value into an `@variable` **without emitting** it. Use the variable later (e.g. in an `httpRequest` body). |
| `encode` | No | Encodes the stored/emitted value, e.g. `encode="json"` to wrap a string in quotes for use in a JSON body. |

### `out` values
| Value | Data type | Description |
|---|---|---|
| `id` | Guid | The Id of the collection. |
| `name` | String | The name of the collection. |

## Examples

Get the name of the current collection:
```xml
<ref:collection out="name"/>
```
Produces: `Test Collection`

Get the Id of the current collection:
```xml
<ref:collection out="id"/>
```
Produces: `10000000-0000-0000-0000-000000000001`

Select a collection by name and return its Id:
```xml
<ref:collection name="Test Collection" out="id"/>
```
Produces: `10000000-0000-0000-0000-000000000001`

Select a collection with a search expression (quotes escaped as `&quot;`):
```xml
<ref:collection search="name = &quot;Test Collection&quot;" out="name"/>
```
Produces: `Test Collection`

Send the collection name to an Azure Function when a rule fires (for example, when a new collection is created). `store` captures the value silently and `encode="json"` wraps it in quotes so it is valid inside the JSON body:
```xml
<ref:collection out="name" store="@newCollection" encode="json"/>
<ref:httpRequest uri="https://YourFunction.azurewebsites.net/api/...">
{
  "Collection created": @newCollection
}
</ref:httpRequest>
```
Produces: the HTTP request body with `@newCollection` resolved to `"Test Collection"`.

## Gotchas
- **`out` is required and case-sensitive.** Valid values are lowercase: `id`, `name`. The lint error also lists `value`, but `out="value"` fails at execution with `Object has no member 'value'` — do not use it.
- **A `@variable` used in `id` must be defined by a preceding `store`.** Using `id="@val"` with no prior `store="@val"` throws at run time (`Variable '@val' does not exist.`); lint warns about it.
- **`store` emits nothing.** A reference with `store=` produces empty output; only an ungated reference (no `store`) prints its value. To both capture and print, use two references or read the variable later.
- **Linting the `store` + `httpRequest` pattern emits a `W098` "declared but never used" warning** for the stored variable. This is a known false positive: the linter does not scan the `httpRequest` body text where the variable is consumed. The references are valid.
- **Concatenating literal text:** raw text outside a reference tag is a parse error. To prefix a label, use sibling `ref:text` references — e.g. `<ref:text>Collection: </ref:text><ref:collection out="name"/>` produces `Collection: Test Collection`. (Nesting a `ref:collection` *inside* a `ref:text` drops the literal prefix.)

## See also
- `../reference/patterns.md` — common reference patterns (store-then-use, httpRequest bodies).
