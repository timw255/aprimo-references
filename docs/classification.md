# Classification (`ref:classification`)

Retrieves a property of a classification — for use in default-value calculations, field validation, rule conditions and actions, and more. Select the classification by Id, by search expression, or use the current classification in scope.

## Syntax
```xml
<ref:classification [id="id" | search="expression"] out="out"/>
```

- By Id: `<ref:classification id="id" out="out"/>`
- By search (must resolve to exactly one classification): `<ref:classification search="expression" out="out"/>`
- Current classification in scope: `<ref:classification out="out"/>`

## Attributes
| Attribute | Required | Description |
|---|---|---|
| `out` | Yes | The property to return (see values below). Case-sensitive. |
| `id` | No | Selects the classification by its Id. May be a literal Guid or an `@variable` defined by a preceding `store`. |
| `search` | No | Selects the classification via a search expression resolving to exactly one classification (e.g. `name = &quot;Acme&quot;`). |
| `store` | No | Captures the value into an `@variable` without emitting it. |

### `out` values
| Value | Data type | Description |
|---|---|---|
| `id` | Guid | The Id of the classification. |
| `identifier` | String | The identifier of the classification. |
| `name` | String | The name of the classification. |
| `label` | String | The label, in the language of the currently logged-in user. |
| `labelPath` | String | The label path, in the language of the currently logged-in user. |
| `namePath` | String | The name path of the classification (e.g. `Brands/Acme`). |
| `parentid` | Guid | The Id of the parent classification. |
| `tag` | String | The value of the Tag property. |
| `createdonlocal` / `createdonutc` | DateTime | Creation date/time, local / UTC. |
| `createdby` | Guid | Id of the user that created it. |
| `modifiedonlocal` / `modifiedonutc` | DateTime | Last-modification date/time, local / UTC. |
| `modifiedby` | Guid | Id of the user that last modified it. |

> **Casing:** most `out` values are lowercase, but `namePath` and `labelPath` are camelCase. The lowercase forms `namepath`/`labelpath` are rejected at lint time.

## Examples

These examples assume a context containing the classification `{ id: "cls-1", name: "Acme", namePath: "Brands/Acme", tag: "premium" }`.

Look up a classification by literal Id and return its name path:
```xml
<ref:classification id="cls-1" out="namePath"/>
```
Produces: `Brands/Acme`

Look up by search expression (quotes escaped as `&quot;`):
```xml
<ref:classification search="name = &quot;Acme&quot;" out="namePath"/>
```
Produces: `Brands/Acme`

Read a classification Id from a field, then look it up — the full **store-then-use** pattern. Here the `Brand` field holds `cls-1`:
```xml
<ref:record fieldName="Brand" store="@brand"/>
<ref:classification id="@brand" out="namePath"/>
```
Produces: `Brands/Acme`

Return the Tag of a classification (useful in rule conditions):
```xml
<ref:classification id="cls-1" out="tag"/>
```
Produces: `premium`

## Gotchas
- **`out` is required and case-sensitive.** Valid set: `createdonlocal, createdonutc, createdby, id, identifier, label, labelPath, modifiedonlocal, modifiedonutc, modifiedby, name, namePath, parentid, tag`. Note the camelCase `namePath`/`labelPath`.
- **A `@variable` in `id` must be defined first** by a preceding `store=`. `<ref:classification id="@val" out="name"/>` with no prior `store="@val"` throws at run time (`Variable '@val' does not exist.`) — pair it with a `ref:record` (or other) `store="@val"` line.
- **`store` emits nothing.** Use a separate, ungated reference to print, or read the variable later.
- **`search` must resolve to exactly one classification.** Supported fields include `name`, `namepath`, and `id`.
- **An id that doesn't resolve throws.** Looking up a classification by an id that isn't found is an error (like `ref:user`/`ref:usergroup` lookups); supply a matching classification in context, or wrap the read in `ref:catch`.

## See also
- `../reference/patterns.md` — store-then-use and field-to-lookup patterns.
- `record.md` — reading the field value that holds the classification Id.
