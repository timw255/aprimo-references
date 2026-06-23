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
- **Loose text is literal — `@variables` are NOT interpolated in it** (only attribute values interpolate). `<ref:text out="42" store="@n"/> value=@n` produces `value=@n`, not `value=42`. To emit a variable, read it with a reference: `<ref:text out="@n"/>`.
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

These check whether each listed variable has been **defined** (ever stored — a variable stored as `""` still counts), **not** whether it has content and **not** whether it is true. List names bare, semicolon-separated. A variable that was *never stored* is treated as absent (false) and does **not** error.

- `onAllVariables="@a; @b"` resolves only when **every** listed variable is defined (AND).
- `onAnyVariable="@a; @b"` resolves when **at least one** is defined (OR).

**Reach for them when the condition spans *several* optional variables.** For a single variable just use `onVariable="IsNotEmpty(@x)"`. But "only if **both** A and B exist" or "if **any** of A/B/C exist" cannot be written with `onVariable` (one predicate) or `eval` (no boolean AND/OR) — that is exactly what these express.

The pieces become optional via a **gated `store`**: a `store` whose gate is false does **not** assign, so the variable stays undefined. So `<ref:text onVariable="IsNotEmpty(@field)" out="@field" store="@v"/>` defines `@v` *only* when `@field` is populated — and `onAllVariables`/`onAnyVariable` later test exactly that.

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
| `onVariable` | a predefined function | Resolve only if the function returns true. See **Gating** above. |
| `onAllVariables` | (variable list) | Resolve only if **all** listed variables are defined (AND of existence). See **Gating** above. |
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

### Alignment (`left` / `right` / `padding`)

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
