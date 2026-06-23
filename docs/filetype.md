# File Type (`ref:filetype`)

Retrieves a property of a file type, selected by Id, name, or search expression.

## Syntax
```xml
<ref:filetype [id="id" | name="name" | search="expression"] out="out"/>
```

## Attributes
| Attribute | Required | Description |
|---|---|---|
| `out` | Yes | The property to return (see values below). Lowercase, case-sensitive. |
| `id` | No | Selects the file type by its Id (Guid). |
| `name` | No | Selects the file type by its name (e.g. `JPEG Image`). |
| `search` | No | Selects the file type via a search expression. Supported fields: `name`, `id`, `extension`. |
| `store` | No | Captures the value into an `@variable` without emitting it. |

### `out` values
| Value | Data type | Description |
|---|---|---|
| `id` | Guid | The Id of this file type. |
| `name` | String | The name of the file type. |
| `extension` | String | The file extension (e.g. `jpg`). |
| `label` | String | The label in the language of the currently logged-in user. |
| `kind` | String | The kind of file type (e.g. `Image`, `Document`). |
| `engineformat` | String | The engine-specific format. |
| `previewformat` | String | The selected preview file format. |
| `previewrequired` | Boolean | `true` when files can only be catalogued if a preview can be created. |
| `iscatalogable` | Boolean | `true` when this file type may be catalogued. |
| `keepdocumentdimensions` | Boolean | `true` when previews keep the original file's dimensions. |
| `tag` | String | The value of the Tag property. |
| `createdonlocal` / `createdonutc` | DateTime | Creation date/time, local / UTC. |
| `createdby` | Guid | Id of the user that created it. |
| `modifiedonlocal` / `modifiedonutc` | DateTime | Last-modification date/time, local / UTC. |
| `modifiedby` | Guid | Id of the user that last modified it. |

## Examples

Examples below resolve against a default Aprimo DAM setup (PDF, JPEG, PNG, generic Image file types).

Get the preview format used for PDFs:
```xml
<ref:filetype name="Portable Document Format (PDF)" out="previewformat"/>
```
Produces: `Jpg`

Get the kind of a file type:
```xml
<ref:filetype name="JPEG Image" out="kind"/>
```
Produces: `Image`

Get the file extension:
```xml
<ref:filetype name="JPEG Image" out="extension"/>
```
Produces: `jpg`

Check a boolean property (returned as the string `true`/`false`):
```xml
<ref:filetype name="JPEG Image" out="iscatalogable"/>
```
Produces: `true`

Get the Tag of a file type:
```xml
<ref:filetype name="JPEG Image" out="tag"/>
```
Produces: `image`

## Gotchas
- **`out` is required and lowercase/case-sensitive.** Valid set: `id, name, extension, label, kind, engineformat, previewformat, previewrequired, iscatalogable, keepdocumentdimensions, tag, createdonlocal, createdonutc, createdby, modifiedonlocal, modifiedonutc, modifiedby`.
- **Boolean outputs render as the strings `true`/`false`**, not `True`/`False`.
- **`name`/`search` lookups must match exactly** (case-insensitive). Use the full file-type name, e.g. `Portable Document Format (PDF)`, not `PDF`.
- **`store` emits nothing.** Capture into an `@variable` for later use; use a separate ungated reference to print.

## See also
- `../reference/patterns.md` — lookup and store-then-use patterns.
