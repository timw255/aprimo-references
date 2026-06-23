# Field reads (field-selecting attributes)

A "field" is not a standalone reference — you read a field by adding **field-selecting attributes** (`fieldName`, `fieldID`, or `field`) to a reference that supports fields, most commonly [`ref:record`](record.md). These attributes pick a field, and `out` says what to return about it (its value, name, label, or a type-specific projection). This page documents those attributes; the host tag (`ref:record`) is a leaf when used this way.

## Syntax
```xml
<ref:record fieldName="FieldName" [languageName="..."|languageID="..."] [out="value"] />
<ref:record fieldID="field-guid" [out="value"] />
<ref:record field="current" [out="value"] />
```

## Attributes
| Attribute | Description |
|---|---|
| `fieldName` | Select the field **by name**. Case-sensitive (`fieldName`, not `fieldname`). |
| `fieldID` | Select the field **by id**. Note the casing: **`fieldID`**, not `fieldId`. |
| `field` | `field="current"` selects the field being validated/defaulted in a field-context rule (validation routines, default values). |
| `languageName` / `languageID` | For multilingual fields, the language whose value is returned. Casing: **`languageID`**. Defaults to the current user's language. |
| `out` | What to return about the field (see values below). Defaults to `value`. |

### `out` values
| Value | Type | Description |
|---|---|---|
| `value` | (depends) | The field's value in its own datatype. **Default.** Honors `languageName`/`languageID`; multi-value fields return a collection. |
| `name` | String | Name of the field. |
| `label` | String | Label of the field in the current user's language. |
| `tostring` | String | String representation of the value. |
| `valuename` | String | (Option List) Name of the selected value(s). |
| `valuechildren` / `valueparents` / `valuelinks` | collection | (Record Link) Related record values. |
| `localvalue` / `utcvalue` | DateTime | (DateTime fields) Value in local time / UTC. |
| `year` / `month` / `day` | Int | (Date fields) Parts of the stored date. |

## Examples

**Read a field's value** (the default — `out="value"` is optional):
```xml
<ref:record fieldName="Description" out="value"/>
```
Produces: `"Hello world"` (context: `{"records":[{"fields":{"Description":"Hello world"}}]}`)

**Read the field's name instead of its value:**
```xml
<ref:record fieldName="Description" out="name"/>
```
Produces: `"Description"` (same context)

**Coerce a value to its string form** with `out="tostring"`:
```xml
<ref:record fieldName="Rating" out="tostring"/>
```
Produces: `"5"` (context: `{"records":[{"fields":{"Rating":"5"}}]}`)

**Reference the current field in a field-context rule.** In a validation routine or default-value rule, `field="current"` points at the field being processed without naming it — so the rule works on whatever field it is attached to:
```xml
<ref:record field="current" out="value"/>
```
Produces: `"Test Value"` (against the default context's current field)

```xml
<ref:record field="current" out="name"/>
```
Produces: `"TestField"` (same)

**Read a multilingual field in a specific language** (names a record, so `id` is supplied):
```xml
<ref:record id="07848ebe-58c7-405b-810a-a8f9178bebf0" fieldName="Description" languageName="French" out="value"/>
```

## Gotchas

- **Casing is enforced.** `fieldName`/`fieldID`/`languageID`/`languageName` are the exact spellings; `fieldname`, `fieldId`, `languageId` are all rejected at lint time.
- **A missing field reads as empty.** Selecting a field that doesn't exist (`fieldName="Nope"`) returns `""`, just like an empty-but-present field — it is not an error. A reference reading a missing field still runs, so you don't need `ref:catch` for that.
- **`label` falls back to the name** when no separate label is defined in context.
- **Multi-value fields return a collection.** It auto-joins with commas in a text context; to control the separator or read per-item properties, feed it to [`ref:foreach`](foreach.md).
- **You can't `<ref:field>`.** There is no standalone field tag — these attributes live on a host reference such as `ref:record`.

## See also
- [`record.md`](record.md) — the primary host for field reads, plus record/file properties.
- [`../reference/operators-and-logic.md`](../reference/operators-and-logic.md) — comparing and gating field values.
