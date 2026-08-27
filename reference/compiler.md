# The Aprimo compiler (transpiler)

Bundled in `bin/`: use `transpiler.exe` on Windows, `transpiler` on macOS, `transpiler-linux` on
Linux (Claude's sandbox on claude.ai is Linux x86-64). Substitute that path wherever the examples
below say `transpiler.exe`. The reference XML is passed on **stdin**; results print to **stdout**.
This is your oracle — it decides whether a reference is valid and what it produces. Trust it over
memory.

## Modes (`-mode`)

| Mode | Purpose | Output |
|---|---|---|
| `lint` | Validate without running. **Use first, always.** | `{"valid":bool,"warnings":[...]}` |
| `execute` | Run the reference and return its output | text after `=== Execution Result ===` |
| `debug` | Run with instrumentation and dump variable state (final, or at `-break N`) | `{"line","variables","callStack","done","error"}` |
| `analyze` | Structural report (depth, branches, loops, tag counts) | JSON |
| `dump-context` | Print the full default execution context as JSON | JSON |

## Flags

- `-mode <mode>` — operation (default `execute`)
- `-context-json '<json>'` — execution context as an inline JSON string (for `execute`)
- `-context <file>` — execution context from a file
- `-i <file>` / `-o <file>` — read input / write output from files instead of stdin/stdout
- `-batch` — read many inputs from stdin, one per line (for bulk `lint`)
- `-break <N>` — for `-mode debug`: pause at XML line `N` (default `0` = run to the end)

## LINT — do this first

```bash
echo '<ref:record fieldName="Title" out="value" store="@title"/><ref:text out="@title"/>' | bin/transpiler.exe -mode lint
# -> {"valid":true,"warnings":[]}
```

When it fails, the message is specific and actionable — it usually hands you the fix:

```bash
echo '<ref:compare operator="in" value1="@xv" value2="5"/>' | bin/transpiler.exe -mode lint
# -> {"valid":false,"warnings":[{"message":"ref:compare has invalid operator='in'
#     (valid: equal, eq, ==, notEqual, ne, !=, greaterThan, gt, greaterThanOrEquals, ge,
#      lessThan, lt, lessThanOrEquals, le)", ... ,"severity":"error"}]}
```

Read the message, apply the named fix, lint again. Repeat until `"valid":true`.

## EXECUTE — confirm behavior

Lint proves it's *valid*; execute proves it's *correct*. A reference can compile and still do the
wrong thing.

```bash
# default context
echo '<ref:text out="Hello"/>' | bin/transpiler.exe -mode execute
# -> "Hello"

# your own data, to exercise the logic
echo '<ref:record fieldName="Status" out="value" store="@status"/><ref:compare operator="eq" value1="@status" value2="Approved" store="@ok"/><ref:text onVariable="IsNotZero(@ok)" out="LIVE"/><ref:text onVariable="IsZero(@ok)" out="DRAFT"/>' \
  | bin/transpiler.exe -mode execute -context-json '{"records":[{"fields":{"Status":"Approved"}}]}'
# -> "LIVE"
```

**Test every branch.** Run the same reference with different `-context-json` values so you see the
output flip — e.g. `Status` = `"Approved"` vs `"Draft"`, or a recent vs old `createdOnUTC`. If the
output doesn't change when the input should change a branch, the logic is wrong even though it
compiled.

### The context shape

`-mode dump-context` prints the **complete default context** as JSON — every key the runtime
understands, pre-populated with sample data (records, users, classifications, languages, file
types, user groups, counters, globals, settings, …). `bin/default_context.json` is a saved copy of
that same output. Start from it, change the parts your reference touches, and pass it back with
`-context-json '<json>'` or `-context <file>`.

List/typed fields have a specific format in the context (Option List = structured `{value,
valuename}`; Classification/User/Record List = an id + a sibling lookup table; Text List = an
array), and each needs its `dataType` declared in `fieldMetadata`. The default context has one
worked example of every type — see [`../docs/field.md`](../docs/field.md) (*Representing fields in
the execution context*) rather than guessing.

You only need to supply the keys you care about — anything you omit falls back to the default. A
context you pass **replaces** the default wholesale, though, so include every record/user/etc. your
reference needs (it is not merged key-by-key with the built-in default).

This is the one place that documents how to populate the context; `dump-context` is the live source
of truth for the exact shape, and each tag's own `docs/*.md` names the key it reads (e.g.
`setting.md` → `settings`, `dict.md` → `dictionary`) with a tag-specific example.

#### Modifying the context

The pieces you'll vary most (verified shapes):

- `records[0].fields.<FieldName>` — what `fieldName="X"` reads (e.g. `{"fields":{"Status":"Approved"}}`).
  The default record starts with an empty `fields` map, so add the fields your reference reads.
- `records[0].contentType`, `.status`, `.createdOnUTC`, `.createdOnLocal`, `.tag` — record
  attributes via `out="contenttype"`, `out="status"`, etc.
- `records[0].files.master.filesize` — what `file="master" out="filesize"` reads
  (e.g. `{"files":{"master":{"filesize":12000000}}}`).
- `settings`, `dictionary`, `collection`, `counters`, `globals` — name/value maps for those tags.

Example exercising several at once:
```json
{"records":[{"contentType":"Print Advertisement","status":"Approved",
  "fields":{"PrintResolution":"300","ExpirationDate":"2027-06-01"},
  "files":{"master":{"filesize":12000000}}}]}
```

#### Adding things to search for

Each searchable reference resolves against a specific array in the context. To make
`search=`/`name=`/`id=` find something, add an entry to the matching array. **Search matching is a
simplified simulation, not the real Aprimo query engine** — the accepted syntax differs per tag:

| Reference | Context array | How `search=` matches | Minimum entry shape |
|---|---|---|---|
| `ref:classification` | `classifications` | `field = 'value'` where field is `namepath`, `name`, or `id` (exact, case-sensitive value) | `{"id","name","namepath"}` (also `label`,`labelPath`,`parentid`,`tag`,timestamps for `out=`) |
| `ref:record` | `records` | case-insensitive **substring** match against the record's `id`, `status`, `createdBy`, and any field value | `{"fields":{...},"status":...}` |
| `ref:user` | `users` | case-insensitive **substring** match against `id`, `name`, `username`, `email` | `{"id","name","username","email"}` |
| `ref:usergroup` | `userGroups` | `field = value` where field is `name` or `id` | `{"id","name"}` |
| `ref:language` | `languages` | `field = value` where field is `name`, `id`, or `culture` | `{"id","name","culture"}` |
| `ref:filetype` | `fileTypes` | `field = value` where field is `name`, `id`, or `extension` | `{"id","name","extension"}` |
| `ref:collection` | `collection` (single object, not an array) | by `name`/`id` lookup against the one collection in context | `{"id","name"}` |

Examples (run against the default context unless you override it):
```bash
# classification by namepath
echo "<ref:classification search=\"namepath = 'Content/Marketing'\" out=\"id\"/>" | bin/transpiler.exe -mode execute
# -> "class-marketing-001"

# user by name/username/email substring
echo '<ref:user search="Administrator" out="email"/>' | bin/transpiler.exe -mode execute
# -> "admin@example.com"

# add your own classification, then resolve it
echo "<ref:classification search=\"name = 'Legal'\" out=\"id\"/>" \
  | bin/transpiler.exe -mode execute -context-json '{"classifications":[{"id":"cl-legal","name":"Legal","namepath":"Content/Legal"}]}'
# -> "cl-legal"
```

> **Variables in a search string.** `@name` inside any attribute (including `search=`) is substituted
> before the lookup runs (e.g. `search="namepath = '/Sec/@status'"`). Because `@` is the variable
> sigil, a literal `@` followed by a letter is read as a variable — so a raw email like
> `search="alice@example.com"` tries to read variable `@example` and fails validation. Search by the
> local part or another field instead, or store the address in a variable first.

`-mode dump-context` always prints the live default if you need the exact current shape of any of
these arrays.

## DEBUG — inspect the variables

When the output is wrong but you can't see *why*, `-mode debug` runs the reference with
instrumentation and dumps the value of **every variable** — no need to bracket them by hand. By
default it runs to the end and captures every variable's final value:

```bash
echo '<ref:record fieldName="X" out="value" store="@xv"/><ref:compare operator="gt" value1="@xv" value2="5" store="@big"/>' \
  | bin/transpiler.exe -mode debug -context-json '{"records":[{"fields":{"X":"10"}}]}'
# {
#   "line": 999999,
#   "variables": { "xv": "10", "big": 1, "__temp1": "" },
#   "callStack": [],
#   "done": true,
#   "error": ""
# }
```

Read it directly: `@xv` resolved to `"10"`, the `gt` comparison stored `big: 1` (true). If a value
is empty, `0`, or missing here, that's your bug. Notes:

- Variables appear **without** the `@` prefix; internal temporaries show as `__tempN`.
- A real runtime error lands in the `"error"` field (the run still reports what it captured).
- `-break <N>` pauses at XML line `N` instead of the end — useful to see state *partway* through a
  long reference (it captures when execution reaches that line).

## Bulk lint

```bash
printf '<ref:text out="a"/>\n<ref:bogus/>\n' | bin/transpiler.exe -mode lint -batch
# one JSON verdict per input line
```

## Reading output in `execute`

Output appears after the `=== Execution Result ===` marker. A leading line like
`Using default context` / `Using provided context JSON` is informational. The returned value is the
quoted string the reference produced — that's what the end user would see in Aprimo.
