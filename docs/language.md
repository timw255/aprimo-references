# Language (`ref:language`)

Retrieves a property of a language, selected by Id, name, or search expression.

## Syntax
```xml
<ref:language [id="id" | name="name" | search="expression"] out="out"/>
```

## Attributes
| Attribute | Required | Description |
|---|---|---|
| `out` | Yes | The property to return (see values below). Lowercase, case-sensitive. |
| `id` | No | Selects the language by its Id. |
| `name` | No | Selects the language by its name (e.g. `French (France)`). |
| `search` | No | Selects the language via a search expression. Supported fields: `name`, `id`, `culture`. |
| `store` | No | Captures the value into an `@variable` without emitting it. |

### `out` values
| Value | Data type | Description |
|---|---|---|
| `id` | Guid | The Id of this language. |
| `name` | String | The name of the language. |
| `culture` | String | The culture the language is linked to (e.g. `en-US`). |
| `enabledforfields` | Boolean | `true` when the language is enabled for field values. |
| `tag` | String | The value of the Tag property. |
| `createdonlocal` / `createdonutc` | DateTime | Creation date/time, local / UTC. |
| `createdby` | Guid | Id of the user that created it. |
| `modifiedonlocal` / `modifiedonutc` | DateTime | Last-modification date/time, local / UTC. |
| `modifiedby` | Guid | Id of the user that last modified it. |

## Examples

Examples below resolve against a default Aprimo DAM setup (English (United States), Spanish (Spain), French (France), German (Germany)).

Get the culture for a language by name:
```xml
<ref:language name="French (France)" out="culture"/>
```
Produces: `fr-FR`

Find a language by culture and return its name:
```xml
<ref:language search="culture = en-US" out="name"/>
```
Produces: `English (United States)`

Find a language by its full name and return the culture:
```xml
<ref:language search="Name = Spanish (Spain)" out="culture"/>
```
Produces: `es-ES`

Check whether a language is enabled for field values (boolean rendered as a string):
```xml
<ref:language name="English (United States)" out="enabledforfields"/>
```
Produces: `true`

Get the Tag of a language:
```xml
<ref:language name="French (France)" out="tag"/>
```
Produces: `fr-FR`

## Gotchas
- **`out` is required and lowercase/case-sensitive.** Valid set: `id, name, culture, enabledforfields, tag, createdonlocal, createdonutc, createdby, modifiedonlocal, modifiedonutc, modifiedby`. The capitalized `out="Culture"` (as shown in some source docs) is **rejected** — use `out="culture"`.
- **`search` matches the value exactly** (case-insensitive on the whole value). `search="Name = English"` does **not** match `English (United States)` and fails at runtime with `no language matches search expression`. Search by `culture` (e.g. `culture = en-US`) or by the full name.
- **Boolean outputs render as the strings `true`/`false`.**
- **`store` emits nothing.** Capture into an `@variable` for later use; use a separate ungated reference to print.

## See also
- `../reference/patterns.md` — lookup and store-then-use patterns.
- `translation.md` — translating keys into a chosen language (`languageId`).
