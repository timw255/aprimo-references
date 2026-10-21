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
| `search` | No | Selects the classification via a search expression that must resolve to **exactly one** classification (e.g. `namepath = '/Brands/Acme'`). Use single quotes inside the attribute. |
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

Look up by search expression. Write the value in **single quotes** inside the attribute
(`&quot;` also works on the platform, but it breaks when the reference is pasted through chat
or an editor that un-escapes it):
```xml
<ref:classification search="namepath = '/Brands/Acme'" out="id"/>
```
Produces: `cls-1`

The search accepts the path **with or without** the leading slash, with or without a trailing
slash, and in any case - all four of these resolve the same classification:
`'/Brands/Acme'`, `'Brands/Acme'`, `'/Brands/Acme/'`, `'/brands/acme'`. The **value** that
`out="namePath"` returns, on the other hand, **always starts with a slash** (`/Brands/Acme`), so
a `compare`/`switch` against a namePath you read back must include it.

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
- **`search` must resolve to exactly one classification.** Searchable properties:
  `name`, `namepath`, `labelpath`, `identifier`, `id`, and the parent subquery `parent.name` /
  `parent.id`. Property names fold case; path values match with or without the leading slash.
  The three failure modes are all **catchable throws**, never an empty result:
  - no match: `No classification has been found matching the specified criterion. {0}` (the `{0}`
    is the platform's own text) - a namepath search for a node that does not exist **fails the
    whole reference** unless wrapped in `ref:catch`;
  - several matches: `More than one matching item has been found.` (so `name = 'Approved'` throws
    when two branches have an *Approved* node - search by `namepath`, not `name`);
  - unknown property: `Error occurred processing the property 'FieldLabel'. The specified field
    'xyz' does not exist.`
- **An empty id throws "The specified value is not a Guid."** This is what you get from `id=""`,
  from `id="@var"` where `@var` holds an empty string or a **blank Classification List field**,
  and from a variable holding a **multi-value** list (`"guid1, guid2"` is not a Guid). Gate the
  lookup with `onVariable="IsNotEmpty(@id)"` and iterate multi-value fields with `ref:foreach`.
- **An id that doesn't resolve throws** (`No classification has been found matching the specified
  criterion. {0}`), like `ref:user`/`ref:usergroup` lookups; supply a matching classification in
  context, or wrap the read in `ref:catch`.
- **`out="tostring"` is not a classification projection** - it throws `Attribute 'out' has an
  invalid value 'tostring' for node 'classification'.`. `tostring` lives on `ref:record`
  (see [field.md](field.md)), where it returns the classification **name** for a Classification
  List field.
- **`search="name = @var"`** works unquoted (`@var` is substituted into the expression), but a
  bare name is ambiguous across branches; prefer `namepath = '@path'`.

## Setting a Classification List field's Default Value

A Classification List field **can** take a Default Value reference (do not route this through a
"Link classification" rule unless you also need the record linked). The reference must return the
classification's **ID** - a namepath string is not accepted. The working form:

```xml
<ref:classification search="namepath = '/Security/Published/Restricted'" out="id"/>
```

For a default that computes the path from other fields, build the path, look it up, and gate it so
the lookup never runs on an empty input (a miss throws and the field stays blank):

```xml
<ref:record fieldName="Status" out="tostring" store="@status"/>
<ref:object onVariable="IsNotEmpty(@status)">
  <ref:classification search="namepath = '/Security/Published/@status'" out="id"/>
</ref:object>
```

Set the field's Reset-to-Default trigger to **On New Field** (not **None**, which never fires on a
record) - see [field-configuration.md](../reference/field-configuration.md#default-values-and-when-they-recompute).

## See also
- `../reference/patterns.md` — store-then-use and field-to-lookup patterns.
- `record.md` — reading the field value that holds the classification Id.
