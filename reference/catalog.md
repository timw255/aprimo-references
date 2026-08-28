# Reference tag catalog

Every valid `ref:` tag, grouped by purpose. **Containers** parse child references; **leaves** are
self-closing (or take only text/attributes). Anything not on this list is not a real tag — if you
reach for `ref:if`, `ref:and`, `ref:else`, `ref:set`, stop: see `operators-and-logic.md`.

For full attributes and behavior of any tag, read its file in `docs/` (named in each row). When in
doubt, read the doc — then verify with the compiler.

## Output & values (the building blocks)

| Tag | Kind | Purpose | Key attributes | Doc |
|---|---|---|---|---|
| `text` | container | Emit literal text and/or variables; group children | `out`, `store`, `format`, `onVariable` | text.md |
| `record` | leaf | Read a field or record attribute | `fieldName`, `out`, `store`, `file` | record.md |

> `ref:record` reads three kinds of thing via `out`: a **field** value
> (`fieldName="X" out="value"`), a **record attribute** (`out="contenttype"`, `out="status"`,
> `out="createdonutc"`), or a **master-file property** (`file="master" out="filesize"` /
> `out="basefilename"` / `out="filename"`). Use `file="master"` (or `file="current"` for
> file-triggered rules) for file reads. `out="size"` is NOT valid — it's `out="filesize"`.
| `field` | leaf | Read the current field's properties | `out`, `store` | field.md |
| `setting` | leaf | Read a system/configuration setting | `name`, `out`, `store` | setting.md |
| `dict` | leaf | Look up a dictionary/term value | `key` or `name`, `out`, `store` | dict.md |
| `count` | leaf | Count items in a collection/field | `store`, target attrs | count.md |
| `eval` | leaf | Evaluate an expression over variables | `expression`, `store` | eval.md |
| `increment` | leaf | Increment a counter | `store`, counter attrs | increment.md |
| `random` | leaf | Random number | `minimum`, `maximum`, `store` | random.md |

## Logic & control flow

| Tag | Kind | Purpose | Key attributes | Doc |
|---|---|---|---|---|
| `compare` | leaf | Compare two values → stores 1/0 result | `value1`, `value2`, `operator`, `store` | compare.md |
| `switch` | container | Multi-way branch; children are `<item key="...">` and `<default>` | `keys` | switch.md |
| `foreach` | container | Loop a collection, emit per item | `in`, `storeitem`, `join` | foreach.md |
| `catch` | container | Run children, fall back to `out` on error | `ex`, `out` | catch.md |
| `object` | container | Group/scope children (often for gating a block) | `onVariable`, `onAllVariables`, `onAnyVariable` | object.md |

> Branching/gating is done with `switch`, or with the gate attributes `onVariable` /
> `onAllVariables` / `onAnyVariable` on `text`/`object`. See `operators-and-logic.md`.

## Dates & time

| Tag | Kind | Purpose | Key attributes | Doc |
|---|---|---|---|---|
| `now` | leaf | Current datetime | `format`, `store` | datetime.md |
| `date` | leaf | Current date | `format`, `store` | datetime.md |
| `datetime` | leaf | Datetime value | `format`, `store` | datetime.md |
| `datetimeeval` | leaf | **Date math** — add/subtract time | `in`, `addYears`, `addMonths`, `addDays`, `addHours`, ... | datetimeeval.md |
| `datediff` | leaf | Difference between two dates | `date1`, `date2`, `timeunit`, `store` | datediff.md |

## Strings

| Tag | Kind | Purpose | Key attributes | Doc |
|---|---|---|---|---|
| `replace` | leaf | Replace substring (e.g. strip whitespace) | `in`, `oldValue`, `newValue`, `store` | replace.md |
| `regex` | leaf | Extract a regex match (1-based `match`; no substitution — use `ref:replace`). **THROWS when the pattern matches nothing** — wrap in `ref:catch` | `in`, `expression`, `match`, `store` | regex.md |
| `translation` | leaf | Localized/translated string | `key`, `languageId`, `module`, `store` | translation.md |
| `language` | leaf | Language entity lookup | `id`/`name`, `out`, `store` | language.md |

## Entities & lookups

| Tag | Kind | Purpose | Doc |
|---|---|---|---|
| `user` | leaf | Current user's attributes (`email`, `name`, `groupnames`, ...) | user.md |
| `usergroup` | leaf | User group lookup | usergroup.md |
| `collection` | leaf | Collection lookup/attributes | collection.md |
| `classification` | leaf | Classification lookup (label/path) | classification.md |
| `filetype` | leaf | File type lookup | filetype.md |
| `searchExpression` | leaf | Build a search expression string | searchexpression.md |

## Integration & jobs

| Tag | Kind | Purpose | Doc |
|---|---|---|---|
| `httprequest` | container | HTTP call; children: `request`, `headers`/`header`, `body` | httprequest.md |
| `maintenancejob` | leaf | Maintenance job info (`attempts`, `message`, ...) | maintenance.md |
| `maintenanceaction` | leaf | Maintenance action info | maintenance.md |
| `maintenancetarget` | container | Maintenance target block | maintenance.md |
| `maintenancenotificationerror` | leaf | Maintenance notification error | maintenance.md |
| `orderdeliveredfile` | leaf | Delivered order file info | download.md |
| `ordertargetaction` | leaf | Order target action | maintenance.md |
| `downloadorderinfo` | leaf | Download order info | download.md |
| `uploadinfo` | leaf | Upload info | upload.md |

## Universal attributes

Most leaf tags accept these in addition to their specific ones:

- `store="@var"` — capture the result into a variable for later use
- `out="..."` — which value/attribute to emit
- `format="..."` — a .NET date format on a date value, or `MB`/`KB`/`GB`/`size` on a file size (use `case="upper"`/`"lower"` for case)
- `onVariable` / `onAllVariables` / `onAnyVariable` — gate: only emit if the condition holds
- `onEmpty="..."` — value to emit when the result is empty **or null**; the main guard for a field
  that may be absent (see [Missing values are null](../docs/overview.md#missing-values-are-null))
- `join`, `left`, `right`, `padding`, `encode` — formatting/joining helpers

If the compiler says an attribute is invalid, it lists the ones that tag accepts — use those.
