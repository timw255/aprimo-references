# Debugging references with the compiler

The fastest way to get a reference right is to **run it and look at what it produces** — not to
reason about it. The compiler is bundled (`bin/transpiler.exe` on Windows, `bin/transpiler` on
macOS); reference XML goes on stdin. This guide is the loop to use when a reference is wrong,
blank, or erroring.

## The loop

1. **Lint** (`-mode lint`) — fix every syntax/attribute error first.
2. **See the context** (`-mode dump-context`) — know what data is actually available.
3. **Build a context** that exercises your logic (`-context-json '{...}'`).
4. **Execute** (`-mode execute`) and read the output.
5. **Flip each branch** — change the context so the output *should* change, and confirm it does.
6. **Inspect the variables** (`-mode debug`) when the output is wrong — dump every `@var`'s value
   and find the one that isn't what you expected.
7. If still stuck, **shrink** the reference to the smallest piece that misbehaves, fix, rebuild.

You are done when it lints clean and produces the right output on an input for *every* branch.

## Step 1 — see what the context gives you

`fieldName="X"` reads `records[0].fields.X`; `out="status"` reads a record attribute; `file="master"
out="filesize"` reads a master-file property. To see the real shape and the keys you can populate:

```bash
bin/transpiler.exe -mode dump-context
```

The default record has **no fields** and **no files** — so any field-reading or file-reading
reference returns empty (or errors) until you supply them with `-context-json`.

## Step 2 — build a context that exercises the logic

Populate exactly what your reference reads, with values that drive the branch you want to test.
Verified shapes:

```jsonc
{
  "records": [{
    "fields": { "Status": "Approved", "PrintResolution": "300", "ExpirationDate": "2027-06-01" },
    "status": "Approved",                       // out="status"
    "contentType": "Print Advertisement",       // out="contenttype"
    "createdOnUTC": "2024-01-01T00:00:00Z",     // out="createdonutc"
    "files": { "master": { "filesize": 12000000 } }   // file="master" out="filesize"
  }]
}
```

Every other tag reads its own top-level key (`settings`, `dictionary`, `classifications`, `user`,
…). Don't guess the shape from here: **`dump-context` is the live source of truth**, and
**`compiler.md`** ("Modifying the context" and "Adding things to search for") is the one place that
documents how to populate each key — including the name/value maps and the searchable arrays. Each
tag's own doc (e.g. `../docs/setting.md`) names the key it reads. Override just the parts you need.

## Step 3 — execute and read the output

```bash
echo '<ref:record fieldName="Status" out="value" store="@status"/>
      <ref:compare operator="eq" value1="@status" value2="Approved" store="@ok"/>
      <ref:text onVariable="IsNotZero(@ok)" out="LIVE"/>
      <ref:text onVariable="IsZero(@ok)"   out="DRAFT"/>' \
  | bin/transpiler.exe -mode execute -context-json '{"records":[{"fields":{"Status":"Approved"}}]}'
# => "LIVE"
```

The output is the quoted string after `=== Execution Result ===` — exactly what Aprimo would show.

## Step 4 — flip every branch

Run the **same** reference with a context that should take the *other* path, and confirm the output
changes:

```bash
... -context-json '{"records":[{"fields":{"Status":"Draft"}}]}'
# => "DRAFT"
```

If the output does **not** change when the input should change a branch, the logic is wrong even
though it compiled. Common causes: the gate never fires, a `switch` key doesn't match exactly, or a
value isn't being read (see the table below).

## Step 5 — see every variable (`-mode debug`)

Before bracketing things by hand, just dump every variable's value with `-mode debug`. It runs the
reference instrumented and reports the final value of *every* variable:

```bash
echo '<ref:record fieldName="X" out="value" store="@xv"/><ref:compare operator="gt" value1="@xv" value2="5" store="@big"/>' \
  | bin/transpiler.exe -mode debug -context-json '{"records":[{"fields":{"X":"10"}}]}'
# "variables": { "xv": "10", "big": 1, ... }
```

Now you can *see* where it went wrong: a variable that's empty, `0`, or missing when it shouldn't be
is the bug. Variables appear without the `@`; internal temporaries are `__tempN`; a runtime error
shows in the `"error"` field. Use `-break <N>` to capture state partway through a long reference.

## Step 6 — shrink to isolate

If `-mode debug` doesn't make it obvious, cut the reference down to the smallest piece that's wrong:

- Replace a suspect sub-expression with a literal and re-run — does the rest work?
- Pull one `compare`/`record` out and run it alone: `<ref:record ... store="@xv"/><ref:text
  out="[@xv]"/>` — and bracket the emit so you can see empty vs whitespace vs a value.

Build back up once each piece is confirmed.

## Reading errors

- **Lint errors are precise** — they name the fix. Invalid operator lists the valid set; unknown
  attribute lists the valid attributes. Reading a variable before it is `store`d is a lint
  **warning** (not a hard error), and a run-time error if reached.
- **Runtime errors** appear in `-mode execute` output. The two you'll hit most:
  - `Variable '@x' does not exist.` — you read a variable that was never `store`d. Define it
    earlier, or wrap the read in `ref:catch` if it may legitimately be unset.
  - `Object has no member 'X'` — usually a wrong-cased `out` value (see below).

  A **missing field** read is *not* an error, but it yields **null** (which renders empty). Passing
  that null on to `ref:switch`, `ref:replace`, `ref:regex`, `ref:count` or `ref:datediff` throws —
  see [Missing values are null](../docs/overview.md#missing-values-are-null). An unknown-entity
  **lookup** — `ref:user`/`ref:usergroup`/`ref:classification` by a bad name/id — also throws.

## Symptom → cause → fix

| Symptom | Likely cause | Fix |
|---|---|---|
| Output is **blank** | Field is empty/absent; or the gate never fired; or the ref only `store`s (no ungated `text`) | Confirm the field via `[ @xv ]` brackets; check the gate predicate; add an ungated `ref:text` to emit |
| Output is blank, but field has data | `out` value wrong-cased (e.g. `out="Id"`, `out="Culture"`) — returns empty at runtime | Use **lowercase** `out` (`out="id"`) |
| `Object has no member 'X'` error | Wrong-cased `out` on a record/collection attribute (`out="Status"`, `collection out="value"`) | Lowercase it (`out="status"`); for collection use `out="id"`/`out="name"` |
| Unknown-entity lookup error | `ref:user`/`ref:usergroup`/`ref:classification` by a name/id that doesn't exist | Verify the name/id; wrap in `ref:catch` if it may be absent (a missing **field** reads empty and needs no catch) |
| Numeric compare always false | The value has **whitespace padding** (` 100 `) — numeric compare fails silently → `False` | Strip it: `ref:replace in="@val" oldValue=" " newValue=""` before comparing |
| Wrong `switch` branch / always default | `switch` keys match **exactly** — `<item key="[0-5]">` matches the literal string, not a range | Use exact keys; for ranges use `compare` + gates |
| `AND` over two compares always true | `onAllVariables`/`onAnyVariable` check whether the variables are **defined**, and a compare always stores `"True"`/`"False"` (both defined) | Use nested `IsNotZero(@xv)` gates (eval cannot combine `True`/`False`) |
| `invalid operator` on compare | Used `contains`/`in`/`>`/`gte`/etc. — not real Aprimo operators | Use the six documented operators; `ref:regex` for substring; compare-per-candidate + OR for membership |
| Date math prints literally (`now+5`) | Used text instead of date math | `ref:datetimeeval in="@now" addYears="5"` |
| `ref:regex` doesn't substitute | `regex`'s `replace` attr is a no-op here — it only extracts a match | Use `ref:replace` for substitution |

## Lint-clean is not enough

Some things pass `lint` but misbehave at **runtime** — always `execute` to be sure:

- Wrong-cased `out` → empty result or an `Object has no member` error.
- `collection out="value"` → crashes at execution.
- `regex replace` → accepted but ignored.

If lint is green but the output is wrong, the answer is almost always in the table above — run it,
bracket the variables, and look.

See also: [`compiler.md`](compiler.md) (full CLI), [`operators-and-logic.md`](operators-and-logic.md),
[`patterns.md`](patterns.md).
