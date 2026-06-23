# Upload reference (`ref:uploadInfo`)

Returns information about a **manual file upload** — who uploaded the file, their language preferences, and the uploaded file name. These references are used in the notifications that Aprimo DAM sends when a user manually uploads a file.

## Syntax

```xml
<ref:uploadInfo out="out" />
```

A single self-closing tag. The `out` attribute selects which piece of information to return.

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `out` | String | Which property of the upload to return (see table below). |

### `out` values

| `out` value | Type | Description |
|---|---|---|
| `creatorEmail` | String | The e-mail address of the user that initiated the upload. |
| `creatorLanguageId` | Guid | The content language ID specified for the user that initiated the upload. |
| `creatorLanguageIdforUi` | Guid | The UI language ID specified for the user that initiated the upload. |
| `fileName` | String | The file name of the file that was uploaded. |

> Common attributes such as `format`, `encode`, `case`, `store`, and the gating attributes (`onVariable`, `onEmpty`, …) work here too. See [overview.md](overview.md).

## Examples

Both examples below lint clean against the compiler. (The default sample context does not populate upload data, so a bare run returns an empty string — these references only carry real values inside an actual upload notification.)

**The uploaded file name:**

```xml
<ref:uploadInfo out="fileName" />
```
Returns the file name of the uploaded file, e.g. `brochure.pdf`.

**The uploader's e-mail** — useful as a notification recipient or for an audit line:

```xml
<ref:uploadInfo out="creatorEmail" />
```
Returns the e-mail address of the user that initiated the upload, e.g. `jane@example.com`.

**A notification sentence** combining the file name and uploader:

```xml
<ref:text out="A new file (" />
<ref:uploadInfo out="fileName" />
<ref:text out=") was uploaded by " />
<ref:uploadInfo out="creatorEmail" />
<ref:text out="." />
```
Produces text such as `A new file (brochure.pdf) was uploaded by jane@example.com.`

## Gotchas

- **Context-bound.** `uploadInfo` only resolves inside a manual-upload notification. Outside that context there is nothing to read, so the values are empty.
- **Language outputs are GUIDs, not names.** `creatorLanguageId` and `creatorLanguageIdforUi` return language IDs. To turn an ID into a readable language name, pass it to a `ref:language` reference — see [language.md](language.md).
- **`store` suppresses output.** Adding `store="@val"` captures the value into `@val` and emits nothing. Only an ungated reference prints. See [overview.md](overview.md).

For more on configuring upload notifications, see the Aprimo "Configuring file upload" documentation.
