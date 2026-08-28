# Regex (`ref:regex`)

Extracts a substring from a value using a regular expression. Pick which match to return with `match` (1-based). This is how you do "contains" / pattern-extraction, which `ref:compare` cannot.

## Syntax
```xml
<ref:regex in="value" expression="pattern" match="1"/>
```

## Attributes
| Attribute | Description |
|---|---|
| in | The text to search (literal or `@variable`). |
| expression | The regular expression — **.NET regex syntax** (see "Supported syntax"). |
| match | (optional) Which match to return, **1-based**. Defaults to `1`. Returns the **whole match** (not a capture group). |
| store | (optional) Variable to capture the result. With `store`, nothing is emitted. |
| replace | Accepted but **ignored** — `ref:regex` only extracts (see Gotchas). |

## Supported syntax

`expression` uses **.NET regular expression syntax**. The full power of .NET regex is available, including features beyond basic patterns:

- **Character classes & shorthand** — `[A-Z]`, `[^abc]`, `\d` `\w` `\s` (and `\D` `\W` `\S`), `.`, `\b` word boundary.
- **Quantifiers** — `*` `+` `?` `{n}` `{n,}` `{n,m}`, and **lazy** variants `*?` `+?` `{n,m}?`.
- **Groups & alternation** — `(...)`, non-capturing `(?:...)`, `a|b`.
- **Named groups & backreferences** — `(?<name>...)`, `\1`, `\k<name>`. (The result is still the whole match, not a single group.)
- **Lookaround** — lookahead `(?=...)` / `(?!...)` and lookbehind `(?<=...)` / `(?<!...)`.
- **Atomic groups** — `(?>...)`.
- **Conditionals** — `(?(expr)yes|no)`.
- **Inline modifiers** — `(?i)` ignore-case, `(?s)` single-line (dot matches newline), `(?m)` multi-line, `(?x)` ignore-pattern-whitespace, and combinations like `(?im)`; also scoped, e.g. `(?i:abc)`.
- **Inline comments** — `(?#a comment)`.
- **Anchors** — `^` `$`, plus the .NET anchors `\A` (start of string), `\Z` / `\z` (end of string), `\G`.
- **Unicode categories** — `\p{L}`, `\p{N}`, `\P{L}`, etc.

## Examples

**Extract the first match** — pull the part number:
```xml
<ref:regex in="AD12345_B1" expression="[A-Z]+[0-9]+" match="1"/>
```
Produces: `"AD12345"`

**Select a later match** — the 2nd match is the text after the underscore:
```xml
<ref:regex in="AD12345_B1" expression="[^_]+" match="2"/>
```
Produces: `"B1"`

**Lookbehind** — the digits *after* `foo` (escape `<` as `&lt;` in XML):
```xml
<ref:regex in="foo123bar" expression="(?&lt;=foo)[0-9]+"/>
```
Produces: `"123"`

**Case-insensitive with an inline flag:**
```xml
<ref:regex in="Status: APPROVED" expression="(?i)approved"/>
```
Produces: `"APPROVED"`

**Store the digits and branch on them:**
```xml
<ref:regex in="img_001.jpg" expression="[0-9]+" match="1" store="@digits"/>
<ref:text out="@digits"/>
```
Produces: `"001"`

## Gotchas
- **`match` is 1-based** and returns the **whole match**, not a captured group. `match="1"` is the first match.
- **No match anywhere → error.** If the expression matches nothing in the input, the reference throws — wrap it in [`ref:catch`](catch.md) if a value may not match. (A `match` index *beyond* the available matches, when the pattern *does* match elsewhere, returns `""`.)
- **The `replace` attribute is a no-op.** `ref:regex` only ever returns a matched substring — it never substitutes. For literal find/replace use [replace.md](replace.md).
- **An empty result is ambiguous — do not use it to mean "no match".** A *successful*
  zero-width match returns `""` (e.g. `expression="o*"` against `alpha`), and so does a
  `match` index beyond the available matches. Since a genuine no-match **throws** instead,
  `IsNotEmpty(@mt)` under-counts any pattern that can match zero characters. This matters
  when the pattern is not under your control — from a setting, say. Seed a sentinel and
  compare against it rather than testing for emptiness:

  ```xml
  <ref:text out="__NOMATCH__" store="@mt"/>
  <ref:catch ex="System.Exception" out=""><ref:regex in="@kw" expression="@pat" store="@mt"/></ref:catch>
  <ref:compare value1="@mt" value2="__NOMATCH__" operator="ne" store="@hit"/>
  ```

- **Escape `<` and `>` as `&lt;`/`&gt;`** inside the attribute (e.g. lookbehind `(?<=...)`, named groups `(?<name>...)`).

See also: [replace.md](replace.md) and [../reference/patterns.md](../reference/patterns.md).
