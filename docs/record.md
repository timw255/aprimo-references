# Record reference (`ref:record`)

Reads a value from a record: one of its **fields**, a **record property** (content type, status, dates, creator), or a property of its **master file**. This is the single most-used reference — almost every rule starts by reading something off the record. It is a **container tag** (its children are parsed and executed, which matters when you read a collection and iterate it).

## Syntax
```xml
<ref:record [id="id"|search="expression"] out="property" />
<ref:record [id="id"] fieldName="FieldName" out="value" />
<ref:record file="master" out="filesize" />
```

The `id`/`search` attribute is **optional** wherever there is a notion of a "current" record (validation routines, default values, field rules) — that is the normal case for rules, and every runnable example below omits it. Everywhere else, supply exactly one of `id` or `search`.

## Attributes
| Attribute | Description |
|---|---|
| `id` | Id of the record to read. Optional when a current record exists. |
| `search` | Search expression identifying the record to read (instead of `id`). |
| `out` | Property to return — see the value tables below. Defaults to `value` for a field read. |
| `fieldName` | Read a field **by name** (e.g. `fieldName="Description"`). Case-sensitive attribute. |
| `fieldID` | Read a field **by id**. Note the casing: **`fieldID`**, not `fieldId`. |
| `field` | `field="current"` selects the field being validated/defaulted in a field-context rule. |
| `file` | Read a property of a file: `master` (use for non-file triggers), `current` (only for file triggers), or a variable holding a file object. |
| `version` | Read a file-version property: `latest`, or a variable holding a file-version object. |
| `additionalFile` | A variable holding an additional-file object, to read additional-file properties. |
| `filter` | When `out` returns a collection (`files`, `versions`, `additionalFiles`), restrict to `new` or `deleted` items (since the last save). |
| `languageName` / `languageID` | Language for a multilingual field value. Note the casing: **`languageID`**, not `languageId`. |
| `iptc` / `exif` / `photoshopInfo` | Return an IPTC / EXIF / Photoshop metadata code instead of `out`. |
| `c2pa` | Return embedded C2PA information (e.g. `c2pa="issuer"`). |
| `key` + `xpath` | With `out="metadata"`, return File-info XML content at the given `key` and XPath. |
| `analysisResult` | Return Aprimo AI data: `description` (text fields) or `tags` (text-list fields). |
| `store` | Capture the value into a variable **without emitting it** (see Gotchas). |
| `format`, `join`, `onVariable`, `onAllVariables`, `onAnyVariable` | Common attributes (formatting, joining a collection, gating). |

### `out` values for the record itself
| Value | Type | Description |
|---|---|---|
| `contenttype` | String | Content type of the record. |
| `status` | String | Status of the record. |
| `id` | Guid | Id of the record. |
| `tag` | String | Value of the Tag property. |
| `createdby` / `modifiedby` | Guid | User that created / last modified the record. |
| `createdonlocal` / `createdonutc` | DateTime | Creation date/time (local / UTC). |
| `modifiedonlocal` / `modifiedonutc` | DateTime | Last-modified date/time (local / UTC). |
| `AIInfluenced` | String | `Yes`, `No`, or `Unknown` (default). |
| `files` | collection | All files in the record (filterable on new/deleted). |
| `versions` | collection | All versions of the master file. |
| `additionalFiles` | collection | All additional files of the latest master version. |

### `out` values when `file=` is set
| Value | Type | Description |
|---|---|---|
| `filename` | String | File name (with extension). |
| `basefilename` | String | File name **without** the extension. |
| `extension` | String | File extension. |
| `filesize` | Long | Size of the file **in bytes**. (There is **no** `out="size"`.) |
| `comment` | String | File comment. |
| `content` | String | Extracted text content of the file. |
| `CRC32` | Int | CRC32 checksum. |
| `versionnumber` / `versionlabel` | Int / String | Version number / label of the latest version. |
| `id` / `recordId` | Guid | Id of the file / its record. |
| `versions` / `additionalFiles` | collection | Versions / additional files of this file. |

(`version=` and `additionalFile=` expose their own smaller `out` sets — `filename`, `id`, `fileId`/`versionId`, `usages`, `createdon`. See the source spec for the full list.)

## Examples

**Read the content type, status, and master-file size.** These are the everyday record properties.
```xml
<ref:record out="contenttype"/>
```
Produces: `"Marketing Asset"` (context: `{"records":[{"contentType":"Marketing Asset"}]}`)

```xml
<ref:record file="master" out="filesize"/>
```
Produces: `"12000000"` (context: `{"records":[{"files":{"master":{"filesize":12000000}}}]}`)

The size comes back in **bytes**. Add `format="MB"` to make it human-readable:
```xml
<ref:record file="master" out="filesize" format="MB"/>
```
Produces: `"11.44 MB"` (same context)

**Read a field value.** Omitting `out` defaults to `value`, so `out="value"` is optional here.
```xml
<ref:record fieldName="Description" out="value"/>
```
Produces: `"A nice product photo"` (context: `{"records":[{"fields":{"Description":"A nice product photo"}}]}`)

**Read a field in a specific language** (multilingual field). Supply `id` because this names a specific record:
```xml
<ref:record id="07848ebe-58c7-405b-810a-a8f9178bebf0" fieldName="Description" languageName="French" out="value"/>
```

**Build a path from the creator's user group.** `store` captures the creator id and group list silently; only the `switch` emits. (Verified with a context that supplies both `records` and `users`.)
```xml
<ref:record out="createdBy" store="@user"/>
<ref:user id="@user" out="groupNames" store="@groups"/>
<ref:switch keys="@groups">
    <item key="Administrators">root/Departments/Admin</item>
    <item key="Designers">root/Departments/Design</item>
    <default>root/Departments/Other</default>
</ref:switch>
```
Produces: `"Admin Path"`-style output. With context
`{"records":[{"createdBy":"...0002"}],"users":[{"id":"...0002","groupNames":["Administrators"]}]}`
this returns `"root/Departments/Admin"`.

**Read image metadata via XPath** (no execution output — depends on embedded File-info XML):
```xml
<ref:record file="master" out="metadata" key="raster" xpath="/raster/pixelsWidth"/>
```

## Gotchas

- **`store` suppresses output.** A `ref:record` with `store="@val"` reads and captures the value but emits **nothing** — even with `out="value"`. This is the key to building logic without polluting output: read a dozen fields into variables, run your compares, and let one ungated `ref:text` emit the final answer. Verified:
  ```xml
  <ref:record fieldName="Title" out="value" store="@title"/><ref:text out="done"/>
  ```
  Produces: `"done"` (the title is captured but not printed).
- **`out="size"` is invalid.** Master-file size is `out="filesize"`. Using `size` is a parse error: `ref:record has invalid out='size' for file context`.
- **Casing matters: `fieldID` and `languageID`** (uppercase ID), not `fieldId`/`languageId`. `fieldName` and `languageName` are also case-sensitive — `fieldname` is rejected at lint time.
- **A missing field reads as empty, not an error.** Reading `fieldName="DoesNotExist"` returns `""`, exactly like a present-but-blank field — so a reference reading a missing field still runs and does not need a `ref:catch`.
- **Record properties are not fields — don't read them with `fieldName`.** The content type, status, id, creator, and dates are record **properties**, read with `out="..."` (`out="contenttype"`, `out="status"`, `out="id"`, `out="createdby"`, …) — *not* `fieldName="..."`. In particular `fieldName="ContentType"` looks for a metadata field named "ContentType" (which usually doesn't exist, so it silently returns `""`); to read the record's content type use `<ref:record out="contenttype"/>`. The compiler warns on `fieldName="ContentType"` for this reason. (`out="contenttype"` returns the content type's **name** as a string; there's no content-type object reference.)
- **`current` file requires a file trigger.** Use `file="master"` for non-file triggers (On New Field, validation, defaults). `file="current"` is only valid for file-related triggers (On Any File Change).
- **Multi-value fields return a collection**, auto-joined with commas only in a text context. To control the separator, iterate with `ref:foreach` and a `join`.

## See also
- [`../reference/patterns.md`](../reference/patterns.md) — the store-then-emit idiom, conditional defaults, crash-safe rules.
- [`../reference/operators-and-logic.md`](../reference/operators-and-logic.md) — comparing and gating the values you read here.
