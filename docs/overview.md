# Working with references

A **reference** is a small XML document. Every XML element in it is resolved into an object, and the whole document collapses down to one final object or string. References power e-mail notification text, field default values, validation routines, file-naming conventions, and any place where one attribute's value is computed from another.

For example:

```xml
My name is <ref:user out="name" />!
```

resolves to:

```
My name is johndoe!
```

(if your user name is `johndoe`).

If a reference returns a non-string object, Aprimo keeps the original data type for as long as it can. A reference that returns a date stays a date until it is concatenated with other text or formatting is applied — only then is it turned into a string.

> **Where references run matters.** Most references work anywhere references are allowed, but some only resolve in a specific context. `ref:maintenance*` references need a maintenance-job context; `ref:downloadOrderInfo` needs a download-order notification; the file-naming variables (`@FILEVERSIONBASENAME`, `@ATTEMPT`, …) only exist while the order agent is naming files. See the relevant tag doc for each one's context.

## Where to go next

| You want to… | Read |
|---|---|
| Translate programming intent (if/else, switch, for-each, AND/OR/NOT, try/catch, recursion) into the exact reference idiom | [`../reference/computational-model.md`](../reference/computational-model.md) |
| See the full list of valid `ref:` tags grouped by purpose | [`../reference/catalog.md`](../reference/catalog.md) |
| Understand logic, gating, and variable flow (the rules the compiler enforces) | [`../reference/operators-and-logic.md`](../reference/operators-and-logic.md) |
| Copy proven, compiler-verified idioms | [`../reference/patterns.md`](../reference/patterns.md) |
| Understand the skill and the bundled compiler | [`../SKILL.md`](../SKILL.md) |

Each individual tag (record, user, foreach, switch, …) has its own page in this `docs/` folder.

## Syntax basics

References must be valid XML. In practice that means:

- They are **case-sensitive** — `fieldName` works, `fieldname` is rejected.
- Every tag is in the **`ref:` namespace** — references start with a `ref:` element (e.g. `ref:text`, `ref:user`, `ref:record`).
- Every attribute value is enclosed in **double quotes** (`"…"`).
- Every element either has a closing tag or self-closes with `/>`.
- There is **no required root element** — a reference can be a flat sequence of sibling tags, which is the normal Aprimo style.

A reference can be embedded inside an HTML or XML document, as long as the whole thing still parses as XML.

### References are templates

A reference is a **template**: any literal text outside the `ref:` tags is emitted as-is, and the `ref:` tags are replaced by their resolved values. So you can write text and references together:

```xml
Total: <ref:eval expression="2*5"/> items
```
Produces `Total: 10 items`. This works at the top level and inside containers (`<ref:object>GPS: <ref:eval.../></ref:object>`), and it is how file-naming and email-style references are normally written.

Two rules to remember:
- **Loose text is literal — `@variables` are NOT interpolated in it** (only attribute values interpolate). `<ref:text out="42" store="@num"/> value=@num` produces `value=@num`, not `value=42`. To emit a variable, read it with a reference: `<ref:text out="@num"/>`.
- **`out=` and a body are mutually exclusive.** `<ref:text out="A">B</ref:text>` is rejected — use the attribute *or* a body, not both.

**Whitespace:** each newline, carriage return, and tab in the source renders as **a single space**; runs of literal spaces are preserved as-is; and only the **overall result** is trimmed at its very start and end (containers do not trim their own body). So indented, multi-line references stay readable without injecting stray newlines, but multiple spaces you type on purpose are kept.

### The `out=` pattern

Most leaf references read **one property of one thing** and emit it. The thing is chosen by the tag and its selector attributes (`id`, `search`, `fieldName`, …); the property is chosen by `out`:

```xml
<ref:user out="email" />            <!-- the current user's email -->
<ref:record out="status" />         <!-- the current record's status -->
<ref:record file="master" out="filesize" />
```

`out` is the single most common attribute. Each tag doc lists its valid `out` values.

### Containers vs. leaves

- **Leaf** references are self-closing and emit a value: `ref:text`, `ref:user`, `ref:field`, …
- **Container** references have children that they parse and run, often once per item: `ref:foreach`, `ref:switch`, `ref:object`, `ref:record` (when iterating files/versions), and the maintenance containers.

See [`../reference/catalog.md`](../reference/catalog.md) for which tags are which.

## Variables: `store=` and `@var`

Variables let one reference feed another.

- **Capture** a value by adding `store="@name"` to any reference. The reference **stores the value and emits nothing** (it returns null in output).
- **Read** a variable by writing `@name` in a later attribute. Variables are substituted across *all* attributes, including HTML attributes.

A variable name is **case-sensitive** and:

- must begin with `@` followed by an **alphabetic** character,
- may only contain **alphanumeric** characters (no underscore — an `_` ends the variable name, so `@base_Prev` reads variable `@base` followed by the literal `_Prev`), and

> **Use this on purpose.** Because `_` ends the name, `out="@lic_"` emits the value of `@lic`
> followed by a literal underscore — which is how you build a delimited name without a
> separate `ref:text` for the separator:
>
> ```xml
> <ref:text onVariable="IsNotEmpty(@lic)" out="@lic_"/>
> <ref:text onVariable="IsNotEmpty(@cls)" out="@cls_"/>
> <ref:text out="@seq" right="4" padding="0"/>
> ```
>
> Each gated line contributes `value_` only when its value is present, so absent parts leave
> no stray separators. `.` behaves the same way.
- must be **at least two characters** long — a single letter like `@s` is not a valid name.

```xml
<ref:user out="id" store="@userid" />
<ref:text out="My name is " />
<ref:user id="@userid" out="name" case="upper" />
<ref:text out="." />
```
Resolves to `My name is JOHNDOE.` — verified against the compiler (returns `My name is TEST USER.` with the sample context).

> **A `@var` must be defined by a preceding `store` before you read it.** Reading an undefined variable throws at run time (`Variable '@val' does not exist.`) — a catchable error — and lint warns about it. The exceptions are the predefined **global variables** below, and context-injected variables (like the file-naming ordering variables) which the runtime supplies.

### `store` suppresses emission

This is the key idea behind clean references: **only an ungated reference prints.** A reference with `store=` captures silently.

```xml
<ref:text out="before " />
<ref:text out="captured" store="@val" />
<ref:text out="after" />
```
Resolves to `before after` — the stored value never appears in the output. (Verified.) To actually output a stored value you read it back with `@val`, or, for a collection, iterate it with [`ref:foreach`](foreach.md).

## Gating: resolve only when a condition holds

Gating attributes let a reference resolve conditionally. They are evaluated *before* the reference emits; if the gate is false, the reference is skipped entirely.

| Attribute | Resolves the reference only if… |
|---|---|
| `onVariable` | the named function returns true: `IsEmpty`, `IsNotEmpty`, `IsNull`, `IsNotNull`, `IsZero`, `IsNotZero` (e.g. `onVariable="IsNotEmpty(@val)"`). Takes exactly **one** predicate on **one** variable. |
| `onAllVariables` | **all** listed variables are **defined** (e.g. `onAllVariables="@xv; @yv"`) — an AND of existence. |
| `onAnyVariable` | **at least one** listed variable is **defined** — an OR of existence. |
| `onEmpty` | (different shape) returns the given string when the resolved value is empty. |

```xml
<ref:text out="Hello" store="@Hello" />
<ref:text out="some text" onVariable="IsNotEmpty(@Hello)" />
```
Emits `some text` because `@Hello` is non-empty. (Verified.)

### `onAllVariables` / `onAnyVariable` — gating on whether variables are *defined*

These check whether each listed variable has been **defined** (ever stored — a variable stored as `""` still counts), **not** whether it has content and **not** whether it is true. List names bare, separated by `;` OR `,` (whitespace around names is trimmed; a SPACE is not a separator). A variable that was *never stored* is treated as absent (false) and does **not** error.

- `onAllVariables="@av; @bv"` resolves only when **every** listed variable is defined (AND).
- `onAnyVariable="@av; @bv"` resolves when **at least one** is defined (OR).

> **The tolerance above does NOT extend to `onVariable`.** `onAllVariables`/`onAnyVariable`
> treat a never-stored variable as absent and carry on, but a predicate throws on one:
> `onVariable="IsEmpty(@nope)"` raises `Variable '@nope' does not exist.` and aborts the
> evaluation. `IsNotEmpty`, `IsNull` and the rest behave the same.
>
> So a gate meant to run *when nothing has been stored yet* cannot be written as
> `onVariable="IsEmpty(@acc)"` unless `@acc` was initialised first — seed it with
> `<ref:text out="" store="@acc"/>`, or use `onAllVariables`/`onAnyVariable`, which
> tolerate the undefined case by design.

**Reach for them when the condition spans *several* optional variables.** For a single variable just use `onVariable="IsNotEmpty(@xv)"` — but see the warning above if the variable may never have been stored. But "only if **both** A and B exist" or "if **any** of A/B/C exist" cannot be written with `onVariable` (one predicate) or `eval` (no boolean AND/OR) — that is exactly what these express.

### Gates see what the previous line just stored

Gates are evaluated **in document order, against the current state** — an element's gate
re-reads any variable an earlier element wrote on the same run. The natural
"append if empty, otherwise concatenate" pair therefore fires **both** times:

```xml
<ref:text out="B" store="@pt"/>
<ref:text out="" store="@acc"/>
<ref:text onVariable="IsEmpty(@acc)"    out="@pt"       store="@acc"/>
<ref:text onVariable="IsNotEmpty(@acc)" out="@acc_@pt"  store="@acc"/>
<ref:text out="@acc"/>
```

Produces `"B_B"`, not `"B"`: line 3 stores `B` into `@acc`, and line 4's gate then sees a
NON-empty `@acc` and appends again. Write mutually exclusive branches with
`ref:switch`, or compute the value once and store it once.

A gate that does **not** fire performs **no** `store=` — the variable keeps its previous
value rather than being cleared.

The pieces become optional via a **gated `store`**: a `store` whose gate is false does **not** assign, so the variable stays undefined. So `<ref:text onVariable="IsNotEmpty(@field)" out="@field" store="@val"/>` defines `@val` *only* when `@field` is populated — and `onAllVariables`/`onAnyVariable` later test exactly that.

**Best practice: wrap the optional block in a `ref:object` (or a `ref:text` with children) and gate the container once**, rather than repeating the gate on every child:

```xml
<ref:record fieldName="Latitude"  out="value" store="@latIn"/>
<ref:text onVariable="IsNotEmpty(@latIn)" out="@latIn" store="@lat"/>
<ref:record fieldName="Longitude" out="value" store="@lonIn"/>
<ref:text onVariable="IsNotEmpty(@lonIn)" out="@lonIn" store="@lon"/>

<ref:object onAllVariables="@lat; @lon">
    <ref:text out="GPS: "/><ref:text out="@lat"/><ref:text out=", "/><ref:text out="@lon"/>
</ref:object>
```
Both coordinates present → `GPS: 40.7, -74.0`. Either missing → nothing: the whole block is dropped, and it does **not** error on reading the missing variable. (Verified against the platform.)

**Common gotcha:** a `ref:compare` always stores `"True"` or `"False"` — both *defined* — so gating a compare result with `onAllVariables`/`onAnyVariable` is **always** true. For an actual truth test use `onVariable="IsNotZero(@flag)"`.

```xml
<ref:user out="email" onEmpty="No e-mail address available" />
```
Emits the user's e-mail, or the fallback string if there is none.

For the full semantics (including what counts as null vs. empty), see [`../reference/operators-and-logic.md`](../reference/operators-and-logic.md).

## Reference attributes

Extra attributes change how a reference's result is produced — formatting, datatype conversion, alignment, encoding, conditional resolution, and so on. The full set:

| Attribute | Value | Remarks |
|---|---|---|
| `format` | a character string | Supported values depend on the datatype returned by the reference; an exception is thrown for any other datatype. See **Format details** below. |
| `join` | a character or string | (only useful when the output is a collection) Separator used when the collection is converted to a string. A single character, a multi-character string, or `""` to concatenate with no separator. |
| `left` | a number > 0 | Returns a string of that length with the value left-aligned. |
| `right` | a number > 0 | Returns a string of that length with the value right-aligned. |
| `padding` | a character string | Character used to pad `left`/`right` (default is a space). |
| `encode` | `html` | Escapes the text for safe use in HTML. |
| `encode` | `htmlattribute` | Escapes the text for safe use in an HTML attribute. |
| `encode` | `htmlwithbreaks` | Escapes the text for safe use in HTML, preserving line breaks. |
| `encode` | `json` | Escapes the text into a valid JSON string. Use this for values passed into an `httpRequest` body. |
| `encode` | `none` | No conversion. |
| `encode` | `xml` | Escapes the text for safe use in XML. Note: a literal `&` in the value is not supported. |
| `case` | `upper` / `lower` | Forces upper- or lower-case. |
| `result` | `ref` | Treat the resolved value as a new reference and resolve it. If it isn't valid XML, return it as-is. See **result details**. |
| `result` | `refxml` | Like `ref`, but throws if the resolved value isn't valid XML. |
| `result` | `text` | Default — return the resolved value as-is. |
| `sqlEncoding` | `SQLServer(guid)` | Make the value safe as a SQL Server Guid (errors if not a Guid). |
| `sqlEncoding` | `SQLServer(int)` | Make the value safe as a SQL Server INT (errors if not an int). |
| `sqlEncoding` | `SQLServer(nvarchar)` | Make the value safe as a SQL Server NVARCHAR. |
| `sqlEncoding` | `SQLServer(like)` | Make the value safe as an NVARCHAR used in a LIKE (`*` and `?` are wildcards). |
| `sqlEncoding` | `Oracle(int)` | Make the value safe as an Oracle INT (errors if not an int). |
| `sqlEncoding` | `Oracle(nvarchar2)` | Make the value safe as an Oracle NVARCHAR. |
| `sqlEncoding` | `Oracle(like)` | Make the value safe as an Oracle NVARCHAR used in a LIKE (`*` and `?` are wildcards). |
| `store` | a variable name | Store the resolved value in the variable; the reference returns null. See **store** above. |
| `onVariable` | a predefined function | Resolve only if the function returns true. **Throws if the variable was never stored** — seed it first. See **Gating** above. |
| `onAllVariables` | (variable list) | Resolve only if **all** listed variables are defined (AND of existence). Tolerates a never-stored name. See **Gating** above. |
| `onAnyVariable` | (variable list) | Resolve only if **at least one** listed variable is defined (OR of existence). See **Gating** above. |
| `onEmpty` | (a string) | Return the given string if the resolved value is empty. |

### Format details

`format` values depend on the returned datatype:

- **DateTime** — the MSDN [custom date/time format strings](http://msdn2.microsoft.com/en-us/library/8kb3ddd4.aspx) and [standard date/time format strings](http://msdn2.microsoft.com/en-us/library/az4se3k1.aspx).
- **TimeSpan** — the MSDN [TimeSpan.ToString formats](http://msdn2.microsoft.com/en-us/library/system.timespan.tostring.aspx).
- **Int64** — `format="MB"` (or `KB` / `GB`) renders the number as a size; `format="size"` lets Aprimo pick the best unit automatically.

```xml
<ref:record file="master" out="filesize" format="MB" />
```
Returns `2.44 MB` for a 2,448,454-byte file.

```xml
<ref:record file="master" out="filesize" format="GB" />
```
Returns `8.02 GB` for an 8,024,484,544-byte file.

### join

```xml
<ref:user out="groupnames" join="-" />
```
Returns the user's group names joined by a dash.

### Alignment and truncation (`left` / `right` / `padding`)

`left="N"` and `right="N"` set a value to **exactly N characters**: they pad a
shorter value and **truncate a longer one**. Truncation is not opt-in and is not
what `padding=""` controls - `padding=""` only suppresses the padding of short
values. If you came here looking for "truncate to N characters", this is it; see
the recipe in [`../reference/patterns.md`](../reference/patterns.md).

`left="0"` is rejected (`Attribute 'left' has an invalid value '0'`), the count is
in characters rather than bytes, and a **null** value is skipped entirely - so
`left=` cannot be used to guarantee a fixed width when the field may be missing.

```xml
<ref:text out="test" left="10" />
```
Returns `test______` (six trailing spaces; `_` shown for clarity).

```xml
<ref:text out="test" right="10" />
```
Returns `______test`.

```xml
<ref:text out="test" left="10" padding="" />
```
Returns `test` (empty padding ⇒ no padding).

```xml
<ref:text out="very long test" left="10" padding="" />
```
Returns `very_long_` (truncated to 10).

```xml
<ref:text out="very long test" left="10" />
```
Also returns `very_long_`. Truncation happens with or without `padding=""`.

```xml
<ref:text out="23" right="4" padding="0" />
```
Returns `0023`.

### result details

`result="ref"` makes Aprimo treat the resolved text as a *new* reference and resolve it again. If the field `Source` contains the literal text `<ref:text out="@Username"/>`:

| Reference | Result |
|---|---|
| `<ref:record id="749c…" fieldName="Source" />` | `<ref:text out="@Username" />` |
| `<ref:record id="749c…" fieldName="Source" result="ref" />` | `Administrator` |

If the resolved value isn't valid XML it is returned as-is. `result="refxml"` is the same but throws when the value isn't valid XML.

### Whitespace

**Text and whitespace between elements become part of the output** (references are templates — see
*References are templates* above). Each **newline, carriage return, and tab renders as a single
space**, and runs of literal spaces are kept verbatim — so a multi-line reference inserts one space
wherever there's a line break or tab between elements (including between/around children inside a
`ref:object`, `ref:switch` item, `ref:foreach` body, or `ref:catch`). **Only the overall result is
trimmed at its very start and end** — containers do *not* trim their own body, so `A<ref:object> B </ref:object>C` emits `A B C`.

This means **formatting affects output**: a reference whose pieces are on separate indented lines
emits a space between each piece (the newline + the indent spaces). When you need tight, exact output
(no stray spaces), **put the emitting references on a single line**:

```xml
<!-- multi-line: emits "A B" (the newline between them = one space) -->
<ref:text out="A"/>
<ref:text out="B"/>

<!-- single line: emits "AB" -->
<ref:text out="A"/><ref:text out="B"/>
```

To force *intra*-attribute whitespace to be kept (e.g. literal spaces inside a value), add
`xml:space="preserve"`:

```xml
<ref:text xml:space="preserve"> Hello World </ref:text>
```

## The order the common attributes run in

When several are present on one tag they apply in this order:

`onEmpty` -> `format` -> `left`/`right` (with `padding`) -> `encode` -> `case` -> `sqlEncoding` -> `result`

Two consequences that surprise people:

- **Padding happens BEFORE encoding.** `out="a&b" left="8" padding="."` pads the
  raw 3-character value to `a&b.....` and encodes that, giving `a&amp;b.....` -
  which is 12 characters, not 8. Width is counted on the value, not the output.
- **`case=` applies after `encode=`**, so it upper-cases the entity itself:
  `encode="html" case="upper"` turns `a&b` into `A&AMP;B`.

`left` and `right` both apply when both are given, left first: `left="4"
right="2"` on `abcdefgh` yields `abcd` then `cd`.

## Choosing a gate: `IsEmpty` vs `IsNull`

The two look interchangeable and are not. They differ on exactly the distinction
in the table above:

| Field state | `IsEmpty` | `IsNull` |
|---|---|---|
| has a value | false | false |
| on the record, no value (blank) | **true** | **false** |
| not on the record at all | **true** | **true** |

So `IsEmpty` means "nothing to show" and `IsNull` means "the field is not there".

Use **`IsEmpty`** for almost everything - it is the one that covers a blank field,
which is the ordinary state of anything a user has not filled in. Reach for
`IsNull` only when you specifically need to detect a field that the record does
not carry, for example to tell a genuinely absent field apart from a blank one.

A gate takes exactly one function on one variable. A bare `onVariable="@flag"` is
not a gate, and neither are `onNull`, `onNotNull`, `onZero` or `onNotZero`.

## Which tags take a body

Only `ref:object`, `ref:foreach`, `ref:switch`, `ref:catch` and `ref:text`
*without* `out=` accept child content. Every other tag is a leaf, and giving one a
body throws:

```
Node 'ref:eval' contains nested elements which it does not support.
```

Two details that catch people out:

- **Whitespace counts.** `<ref:eval expression="2">   </ref:eval>` throws just
  like a real child would. Only a body with no character data at all is clean, so
  `<ref:text out="A"></ref:text>` is fine but `<ref:text out="A">   </ref:text>`
  is not.
- **`ref:text` with both `out=` and a body** gets its own message: `Node 'text'
  has at least two conflicting attributes which cannot be used together. One of
  these attributes is 'out'.` Use one or the other. `<ref:text>BODY</ref:text>`
  with no `out=` is a perfectly ordinary template.

## A duplicate attribute stops the reference deploying

Repeating an attribute - `encode="html" encode="json"`, or `out=` and `OUT=`,
which are the same attribute since names fold case - is worse than an error at
run time. **Aprimo refuses to save the reference at all.** The field keeps its
previous default value, the old reference keeps running, and nothing anywhere
reports a problem. The change simply does not take effect.

`-mode lint` reports this as an error. It is the one class of mistake where a
clean-looking deployment is silently the old behaviour.

## Attribute names fold case, and unknown ones are ignored

Attribute names fold case - `out`, `OUT` and `Out` are the same attribute, as are
`storeitem`/`storeItem` and `fieldname`/`fieldName`. Casing is never the problem.

An attribute name no tag recognizes is **ignored**: Aprimo neither warns nor
fails, it runs the reference without it. A typo therefore has no visible symptom
at run time, it simply changes what the reference does:

```xml
<!-- onNull is not an attribute. There is no error, and the text renders
     unconditionally, because no gate was applied. -->
<ref:text out="@value" onNull="fallback"/>
```

The gate attributes are `onVariable`, `onAllVariables` and `onAnyVariable`.

`onEmpty` is **not a gate** — it is a fallback VALUE, substituted when the result is
empty (`out="" onEmpty="FALLBACK"` renders `FALLBACK`; `out="x" onEmpty="FALLBACK"`
renders `x`). It never decides whether the element runs.

`onNotEmpty` is **not an attribute**. Neither are `onNull`, `onNotNull`, `onZero` and
`onNotZero`. All of them are silently ignored, so the element behaves exactly as if the
attribute were absent — the obvious-sounding gate does nothing at all.

`-mode lint` reports every unrecognized attribute as an error, which is the only
way to detect one before it ships.

## Missing values are null - but "blank" usually is not

A field can be missing in two different ways, and they do **not** behave the
same. Getting this backwards is the most common source of a reference that works
in testing and throws on a real record:

| The field is... | Reads as | `count` | `replace` / `regex` | `left=` / `right=` |
|---|---|---|---|---|
| **on the record, no value** (a blank field) | **empty** | `0` | runs, returns empty | pads |
| **not on the record at all** | **null** | throws | throws | skipped |

A blank field on a content class - the normal state of anything a user has not
filled in yet - is **empty**, not null. `IsEmpty` is true for both, so the
ordinary guard covers both cases. The throwing behaviours below apply only to the
second row: a field that is genuinely not there (wrong `fieldName`, a field on a
different content class, a typo).

That is also why `fieldName=` typos are so costly - a misspelled field is not
"blank", it is absent, and the first `ref:count` or `ref:switch` that touches it
throws.

Reading a field the record does not carry is **not** an error and renders
nothing - but the value is **null**, which is not the same as an empty string.
The distinction only shows up when you pass that value on, and then it matters
a lot: some tags dereference it and throw.

**These THROW on a null** (guard them):

| Tag | What happens |
|---|---|
| `ref:switch keys="@val"` | `Attribute 'keys' has an invalid value '' for node 'switch'.` It does **not** fall through to `<default>`. |
| `ref:replace in="@val"` | `Object reference not set to an instance of an object.` |
| `ref:regex in="@val"` | same null-reference error |
| `ref:count in="@val"` | `Attribute 'in' has an invalid value '' for node 'count'.` |
| `ref:datediff date1/date2` | invalid-value error naming the attribute |

**These are safe** — a null behaves like empty: `out="@val"` (renders nothing),
`encode=`, `case=`, `ref:compare` (compares as `""`), `onEmpty=` (fires),
`IsEmpty` (true). `left=`/`right=` and `sqlEncoding=` are **skipped** entirely,
so no padding is applied and no conversion is attempted. A null used as a
`ref:searchExpression` `paramN` leaves its `?` placeholder unsubstituted.

`IsNotNull` is **false** on it, and the `onAllVariables`/`onAnyVariable` gates —
which test for non-null, not merely "defined" — do not treat it as satisfied.

### Is it even a field?

Before guarding, check that the name is a **field** at all. `status`, `contenttype`,
`id`, `createdby`, `createdon*` and `modifiedon*` are record **properties**, read
with `out="..."` - `<ref:record out="status"/>`. Writing `fieldName="Status"` looks
for a metadata field of that name, which usually does not exist, so it reads null.
Guarding that null with `onEmpty="Unknown"` produces a reference that lints clean,
never errors, and is **permanently wrong**. See
[record properties](record.md#record-properties).

### Guarding

Give the read a fallback so the value is never null downstream:

```xml
<ref:record fieldName="Licensetype" out="valuename" onEmpty="Unknown" store="@lic"/>
<ref:switch keys="@lic">
  <item key="Royalty Free">RF</item>
  <default>UNKNOWN</default>
</ref:switch>
```

or gate the whole fragment:

```xml
<ref:record fieldName="Licensetype" out="valuename" store="@lic2"/>
<ref:object onVariable="IsNotEmpty(@lic2)">
  <ref:switch keys="@lic2">...</ref:switch>
</ref:object>
```

An uncaught throw aborts the **entire** reference, so a status badge that works
on records where the field is populated will silently produce nothing on the
records where it is not. This is the most common way a reference that "works"
in testing fails in production.

## Aprimo DAM global variables

These predefined variables can be used in reference attributes without a preceding `store`:

| Variable | Description |
|---|---|
| `@adamUserId` | ID (Guid) of the current user. |
| `@adamUsername` | User name of the current user. |
| `@adamSessionId` | ID (Guid) of the current session. |
| `@adamLanguageIdForUI` | UI language ID (Guid) of the current user. |
| `@adamLanguageId` | Content language ID (Guid) of the current user. |
| `@adamCurrentFieldLanguageId` | Content language ID (Guid) of the user that triggered the event. |
| `@adamRecord` | ID (Guid) of the current record (only in a record context). |
| `@adamObject` | ID of the current object, depending on context (record, user, …). |

> **These names are case-sensitive.**

### Examples

**Current user name:**

```xml
<ref:text out="The current user's name is " />
<ref:text out="@adamUsername" />
```
`The current user's name is Administrator` (when the current user is the Administrator).

**UI vs. content language** (for Spanish user Maria, both set to Spanish, ID `12341234-1234-1234-1234-123412341234`):

```xml
<ref:text out="User UI language:" />
<ref:text out="@adamLanguageIdForUI" />
```
`User UI language:12341234-1234-1234-1234-123412341234`

```xml
<ref:text out="User's field content language:" />
<ref:text out="@adamLanguageId" />
```
`User's field content language:12341234-1234-1234-1234-123412341234`

`@adamCurrentFieldLanguageId` reflects the *last editor's* content language. If the field was last edited by Sandra (English, ID `abcdabcd-abcd-abcd-abcd-abcdabcdabcd`):

```xml
<ref:text out="Last editor's content language:" />
<ref:text out="@adamCurrentFieldLanguageId" />
```
`Last editor's content language:abcdabcd-abcd-abcd-abcd-abcdabcdabcd`
