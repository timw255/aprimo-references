# User Group reference (`ref:usergroup`)

Retrieves a property of a user group, looked up by id, name, or search expression.

## Syntax
```xml
<ref:usergroup id="id" out="out"/>
<ref:usergroup name="name" out="out"/>
<ref:usergroup search="expression" out="out"/>
```

## Attributes
| Attribute | Required | Description |
|---|---|---|
| `out` | yes | The group property to return. Must be one of the lowercase values below. |
| `id` | no | Look up the group by Id. |
| `name` | no | Look up the group by name. |
| `search` | no | Look up the group by a search expression. |

Use at most one of `id`, `name`, or `search`.

### Valid `out` values
The compiler enforces a closed, **lowercase** set. Any other value fails `lint`:

`id`, `name`, `tag`, `createdonlocal`, `createdonutc`, `createdby`, `modifiedonlocal`, `modifiedonutc`, `modifiedby`

| Value | Type | Description |
|---|---|---|
| `id` | Guid | Id of the user group. |
| `name` | String | Name of the user group. |
| `tag` | String | The Tag property. |
| `createdonlocal` / `createdonutc` | DateTime | Creation date/time (local / UTC). |
| `createdby` | Guid | Id of the user that created the group. |
| `modifiedonlocal` / `modifiedonutc` | DateTime | Last-modified date/time (local / UTC). |
| `modifiedby` | Guid | Id of the user that last modified the group. |

## Examples

Get a group's id by name:
```xml
<ref:usergroup name="Everyone" out="id"/>
```
Produces: `"00db70f9-2fd2-417c-b898-337b7a26641a"`

Read the Tag of a group:
```xml
<ref:usergroup name="Designers" out="tag"/>
```
Produces: `"creative_team"`

Look up a group by id and read its name:
```xml
<ref:usergroup id="marketing-guid-789" out="name"/>
```
Produces: `"Marketing"`

Read who created a group:
```xml
<ref:usergroup name="Designers" out="createdby"/>
```
Produces: `"admin-user"`

Capture silently with `store`, then emit:
```xml
<ref:usergroup name="Everyone" out="id" store="@grp"/><ref:text out="@grp"/>
```
Produces: `"00db70f9-2fd2-417c-b898-337b7a26641a"`

## Gotchas
- **`out` values are lowercase and case-sensitive at runtime.** `out="Id"` passes `lint` but resolves to an **empty string** when executed; use `out="id"`. Trigger the lint error with an invalid value (e.g. `out="ZZZ"`) to print the full valid set.
- A lookup that matches no group resolves to an empty string rather than erroring.
- A `store` attribute captures without emitting; pair with an ungated `ref:text` to print. See `../reference/patterns.md`.

## See also
- `user.md` — `out="groupnames"`/`out="groupids"` returns the groups a user belongs to.
- `../reference/patterns.md` — store/emit patterns.
