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
| `value` | (depends) | The field's value in its own datatype. **Default.** Honors `languageName`/`languageID`; multi-value fields return a collection. For **list-type fields** this is the selected object's **id(s)**, not a readable label — see [Reading list fields](#reading-list-fields-the-value-is-an-id-gotcha). |
| `name` | String | Name of the field. |
| `label` | String | Label of the field in the current user's language. |
| `tostring` | String | String form of the value. On a **list-type field this resolves names**: a Classification List renders the classification name(s), a User List the user name(s), an Option List the option name(s) - several items are comma-joined and sorted alphabetically. On a Date field it is the display date. |
| `valuename` | String | (Option List **only**) Name of the selected value(s). On any other type it throws `Attribute 'out' has an invalid value 'valuename' for node 'record'.` |
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

**Read a Classification List field's name in one step** - `tostring` resolves the selected
classification's **name** (not its id, not its path):
```xml
<ref:record fieldName="AssetStatus" out="tostring"/>
```
Produces: `"Approved"` (default-context shape: the field holds a classification id and
`classifications` carries `{id, name: "Approved"}`)

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

## Reading list fields (the value-is-an-id gotcha)

A **list-type field stores the id(s) of the selected object(s)**, not their text (verified
live against Aprimo). So `out="value"` on a list field gives you **ids, not a readable label**
— the single most common surprise when reading fields. How to get the readable value depends
on the type:

| Field type | What `out="value"` returns | Readable in one step | Readable per item (two-step) |
|---|---|---|---|
| **Option List** | the selected option's id | `out="valuename"` or `out="tostring"` (the option name) | - |
| **Classification List** | classification id(s) | `out="tostring"` (the **name**, not the path) | `<ref:classification id="@id" out="name"/>` (or `namePath`/`label`) |
| **User List** | user id(s) | `out="tostring"` (the user name) | `<ref:user id="@id" out="name"/>` |
| **Record List** | record id(s) | - | `<ref:record id="@id" fieldName="..."/>` |
| **Record Link** | **throws** - `value` and `tostring` are invalid on a Record Link | - | `out="valuechildren"`/`"valueparents"`/`"valuelinks"`, then `<ref:record id="@id" .../>` |
| **Text List** | the literal strings (a list) | `out="tostring"` (comma-joined text) | - |

**Valid `out` values per field type** (anything else throws
`Attribute 'out' has an invalid value '...' for node 'record'.`):

| Field type | `value` | `tostring` | `valuename` | `year`/`month`/`day` | `localvalue`/`utcvalue` | `valuechildren`/`valueparents`/`valuelinks` |
|---|---|---|---|---|---|---|
| Text, Multi-line, Html, Number | the text/number | same | throws | throws | throws | throws |
| Date | date | display date (`6/15/2027`) | throws | parts | throws | throws |
| DateTime | date-time | display | throws | parts | yes | throws |
| Option List | id (a list) | option name(s) | option name(s) | throws | throws | throws |
| Classification List | id(s) (a list) | classification name(s) | **throws** | throws | throws | throws |
| User List | id(s) (a list) | user name(s) | throws | throws | throws | throws |
| Record List | id(s) (a list) | - | throws | throws | throws | throws |
| Record Link | **throws** | **throws** | throws | throws | throws | linked record values |
| Text List | the strings (a list) | comma-joined | throws | throws | throws | throws |

`name` and `label` are valid on every type (the field's own name/label, not its value).

**When the reference needs the name, use `tostring`** - it is one step and never throws on a
blank field (it renders empty). A production `Status` switch works exactly this way:

```xml
<ref:record fieldName="Status" out="tostring" store="@status"/>
<ref:switch keys="@status">
  <item key="Expired">...</item>
  <item key="Approved">...</item>
</ref:switch>
```

**When the reference needs the path** (to build or compare a namepath), use the two-step read:
`out="value"` for the id, then `<ref:classification id="@id" out="namePath"/>`. The returned
path **starts with a slash** (`/Security/Published/Restricted`).

**Option List** — one step, no resolution needed:
```xml
<ref:record fieldName="Licensetype" out="valuename"/>   <!-- "Royalty Free", not the id -->
```

**Everything else** — read the id, then resolve the object by that id (loop with `foreach`
for multi-value fields):
```xml
<ref:record fieldName="Status" out="value" store="@statusId"/>
<ref:classification id="@statusId" out="name" store="@status"/>
<ref:text out="@status"/>
```

> **`out="value"` on a list field is a warning, but the wrong *projection* is an error.**
> Reading the id with `out="value"` is fine (it's the first step above) — the compiler just
> warns you'll get ids. But `out="valuename"` on anything other than an Option List, or
> `out="valuechildren"`/`valueparents`/`valuelinks` on anything other than a Record Link,
> **throws a `ReferenceException`**. It's catchable: wrap it in
> `ref:catch` if you need a fallback. (The compiler reproduces this when the field's `dataType`
> is declared in the context — see below.)

## A list field with ONE item is still a list

`out="value"` on any list-type field - Option List, Classification List, User List, Record List,
Text List - returns a **collection even when exactly one item is selected**.
That decides what you can do with the stored variable:

| Consumer | One item selected | Several items |
|---|---|---|
| `ref:count in="@ids"` | `1` | `n` |
| `ref:foreach in="@ids"` | iterates once | iterates n times |
| `ref:classification id="@ids"` / `ref:user id="@ids"` | resolves (the one id) | throws `The specified value is not a Guid.` |
| `ref:switch keys="@ids"` | **throws** `Attribute 'keys' has an invalid value '<id>' for node 'switch'.` | throws |
| `ref:compare value1="@ids" value2="<id>"` | **`False`** (a list never equals a string) | `False` |
| `ref:text out="@ids"`, `left=`, `replace`, `encode=` | the id text | the ids comma-joined |

So never `switch` or `compare` on a list field's `out="value"`. Either read the field with
`out="tostring"` (its name), or copy the ids to plain text first - a `ref:text` hop turns any
field list into its joined string (`<ref:text out="@ids" store="@idText"/>`), after which
`switch`/`compare` behave normally. (A `ref:object store=` keeps the list typed.) The compiler
warns about both when the field's `dataType` is declared in `fieldMetadata`.

**A blank list field reads as NULL through `out="value"`** (so `switch keys=` throws, and
`ref:classification id=` throws "not a Guid"), but as an **empty string through `out="tostring"`**
(so a switch falls through to `<default>`). Prefer `tostring` for anything that branches.

**Multi-value Classification List fields** hold several ids: `out="value"` renders
`"id1, id2"`, `out="tostring"` renders the names alphabetically (`"Deny Delete, Deny Modify"`
regardless of selection order), and only `ref:foreach` lets you resolve each one:

```xml
<ref:record fieldName="Regions" out="value" store="@ids"/>
<ref:foreach in="@ids" storeitem="@id" join="|"><ref:classification id="@id" out="namePath"/></ref:foreach>
```
Produces: `/Assets/Stories/Region/NorthAmerica/USA|/Assets/Stories/Region/NorthAmerica`

## Representing fields in the execution context

To test a reference, the field must be in the context the way Aprimo stores it. **Most field
types are just a string — but the list types are not**, and the compiler can only reproduce the
id-vs-name behavior (and warn you) when the context says so. **Don't guess the format — copy it
from `bin/default_context.json`, which already has one worked example of every type below.**

Two rules:
1. Put the field value under `records[].fields` in the shape for its type (table below).
2. **Always declare the field's type** under top-level `fieldMetadata` (`{"<name>":{"dataType":"<Type>"}}`).
   That declaration is what makes `execute` warn when a read returns ids, and what makes
   Date/DateTime fields read as real dates.

| Field type | `dataType` | Format in `fields` | Needs |
|---|---|---|---|
| Text / Multi-line | `SingleLineText` / `MultiLineText` | `"Title": "Quarterly Report"` | — |
| Number | `Numeric` | `"Rating": 5` | — |
| Date / DateTime | `Date` / `DateTime` | `"Expiry": "2026-12-31"` | declare the type so it reads as a date |
| **Option List** | `OptionList` | `"License": {"value":"opt-id","valuename":"Royalty Free"}` | a **structured value** (id + name) |
| **Classification List** | `ClassificationList` | `"Dept": "cls-id"` | a matching `classifications` entry |
| **User List** | `UserList` | `"Author": "user-id"` | a matching `users` entry |
| **Record List / Record Link** | `RecordList` / `RecordLink` | `"Related": "rec-id"` | a second record in `records` with that `id` |
| **Text List** | `TextList` | `"Keywords": ["a","b","c"]` | — (an array of strings) |

So a field that needs the two-step (Classification/User/Record) is an **id string in `fields`
plus the object it points to in a sibling table** (`classifications` / `users` / `records`) — that
sibling is what `<ref:classification id=...>` / `<ref:user id=...>` / `<ref:record id=...>`
resolves against. Multi-value list fields use an **array of ids** (and, for Option List, arrays:
`{"value":["id1","id2"],"valuename":["A","B"]}`).

A complete Option-List example — note the `fieldMetadata` declaration drives both the warning
and the result:
```bash
echo '<ref:record fieldName="License" out="value"/>' | bin/transpiler.exe -mode execute \
  -context-json '{"records":[{"fields":{"License":{"value":"opt-id","valuename":"Royalty Free"}}}],
                  "fieldMetadata":{"License":{"dataType":"OptionList"}}}'
# stderr: field "License" (OptionList): out="value" returns the selected option's id - use out="valuename"...
# result: "opt-id"            (out="valuename" would give "Royalty Free")
```

A Classification two-step (the field is an id; the name comes from the `classifications` table):
```bash
echo '<ref:record fieldName="Dept" out="value" store="@id"/><ref:classification id="@id" out="name"/>' \
  | bin/transpiler.exe -mode execute -context-json '{
      "records":[{"fields":{"Dept":"cls-mktg"}}],
      "fieldMetadata":{"Dept":{"dataType":"ClassificationList"}},
      "classifications":[{"id":"cls-mktg","name":"Marketing"}]}'
# result: "Marketing"
```

The default context (no `-context-json`) already has `Licensetype` (Option List),
`PrimaryClassification` (Classification List), `Author` (User List), `RelatedAsset` (Record
Link), and `Keywords` (Text List) wired up and resolving — run `-mode dump-context` to see the
whole shape, or just read those fields to see each pattern work.

## Gotchas

- **Casing is enforced.** `fieldName`/`fieldID`/`languageID`/`languageName` are the exact spellings; `fieldname`, `fieldId`, `languageId` are all rejected at lint time.
- **A missing field reads as NULL, which renders empty.** Selecting a field that doesn't exist (`fieldName="Nope"`) is not an error and renders nothing, but the value is **null** — which is *not* the same as an empty-but-present field. Feeding that null to `ref:switch`, `ref:replace`, `ref:regex`, `ref:count` or `ref:datediff` throws. See [Missing values are null](overview.md#missing-values-are-null).
- **`label` falls back to the name** when no separate label is defined in context.
- **Multi-value fields return a collection.** It auto-joins with commas in a text context; to control the separator or read per-item properties, feed it to [`ref:foreach`](foreach.md).
- **You can't `<ref:field>`.** There is no standalone field tag — these attributes live on a host reference such as `ref:record`.

## See also
- [`record.md`](record.md) — the primary host for field reads, plus record/file properties.
- [`../reference/field-configuration.md`](../reference/field-configuration.md) — the field *definition* side: types, default-recompute triggers, scope, storage, and Record-Link traversal (incl. the inverted `valuechildren`/`valueparents` accessors).
- [`../reference/operators-and-logic.md`](../reference/operators-and-logic.md) — comparing and gating field values.
