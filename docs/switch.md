# Switch reference (`ref:switch`)

Selects one output from a list of cases based on a value — the reference equivalent of `Select…Case`. It is a **container tag**: its `<item>` and `<default>` children may hold literal text or nested references. Reach for `switch` when you have a clean value-to-output mapping (asset type → retention years, user group → folder); it is far more readable than a stack of compares.

## Syntax
```xml
<ref:switch keys="@valueOrList">
    <item key="key1">output 1</item>
    <item key="key2">output 2</item>
    <default>fallback output</default>
</ref:switch>
```

## Attributes
| Attribute | Description |
|---|---|
| `keys` | The value(s) to match against item keys. A single value (a field value) or a list (e.g. a user's groups). Usually a variable populated earlier with `store=` (`keys="@type"`); a bare word with no `@` (`keys="Photo"`) is matched as a literal string. The value must be **text** — see Gotchas. |
| `<item key="...">` | Child element. Its content is returned when `key` matches. |
| `<default>` | Optional child element. Its content is returned when no item matches. |

## Examples

**Map a field value to a retention period.** Multiple keys can share an output by repeating the value:
```xml
<ref:record fieldName="AssetType" out="value" store="@type"/>
<ref:switch keys="@type">
    <item key="Photo">5 years</item>
    <item key="Video">5 years</item>
    <item key="Document">3 years</item>
    <default>1 year</default>
</ref:switch>
```
Produces: `"5 years"` (context: `{"records":[{"fields":{"AssetType":"Video"}}]}`)

**Fall through to the default** when nothing matches:
```xml
<ref:record fieldName="AssetType" out="value" store="@type"/>
<ref:switch keys="@type">
    <item key="Photo">5 years</item>
    <default>1 year</default>
</ref:switch>
```
Produces: `"1 year"` (context: `{"records":[{"fields":{"AssetType":"Audio"}}]}`)

**Match against a list of keys.** When `keys` is a **user collection** such as `out="groupNames"`,
the switch matches if any item key is in the list - handy for routing by membership. This does
**not** generalize to record list fields: `keys=` on a TextList/ClassificationList field throws
`Attribute 'keys' has an invalid value '...' for node 'switch'.` For those, iterate with
`ref:foreach`, re-store each item through `ref:text`, and switch on the copy:
```xml
<ref:record out="createdBy" store="@uid"/>
<ref:user id="@uid" out="groupNames" store="@groups"/>
<ref:switch keys="@groups">
    <item key="Designers">root/Departments/Design</item>
    <item key="Administrators">root/Departments/Admin</item>
    <default>root/Departments/Other</default>
</ref:switch>
```
Produces: `"root/Departments/Admin"` (context with creator in the `Administrators` group)

**An item can hold a nested reference**, not just literal text. Escape `&` as `&amp;` in XML:
```xml
<ref:record fieldName="Dept" out="value" store="@dept"/>
<ref:switch keys="@dept">
    <item key="RND"><ref:text out="R&amp;D Department"/></item>
    <item key="MKT"><ref:text out="Marketing Department"/></item>
</ref:switch>
```
Produces: `"R&D Department"` (context: `{"records":[{"fields":{"Dept":"RND"}}]}`)

## Gotchas
- **Key matching is CASE-SENSITIVE.** `keys="ALPHA"` does not match
  `<item key="alpha">` - it falls through to `<default>`. Most of the language
  folds case (attribute names, `out=` values, gate function names); switch keys
  do not. Normalise with `case="lower"` on the value first if you need it.
- **`;` and `,` in `keys=` are NOT separators.** `keys="a;b"` looks for one key
  literally called `a;b`. A switch matches a single whole value.
- **Surrounding whitespace is significant** on both sides: `keys=" a "` does not
  match `<item key="a">`, and neither does the reverse.


- **Keys are EXACT string matches — no ranges, no regex.** `<item key="[0-5]">` matches the literal string `[0-5]`, **not** the numbers 0–5. Verified: a rating of `"3"` against `<item key="[0-5]">in range</item><default>literal-only</default>` produces `"literal-only"`. For numeric ranges, use `ref:compare` with gates instead. A `;` inside a `key` is also literal — it is **not** a multi-value separator.
- **`keys` must be a TEXT value.** Switching directly on the result of `ref:eval` (a number) or `ref:compare` (a boolean) fails — those are typed values, and the switch needs text. Emit the computed value through a `ref:text store="@key"` first, or restructure. A text/field value of `"True"` or `"5"` matches fine; it is only the typed eval/compare output that is rejected.
- **First match wins**, so order your items from most- to least-specific.
- **No match and no `<default>` → empty output.** The switch emits `""`; it does not error.
- **`keys` must be defined earlier in scope** (via `store=`); reading an undefined variable throws at run time (`Variable '@val' does not exist.`).
- **A `store` inside an item DOES escape the block** - there is no block scope. It holds whatever the item that actually ran assigned; an item that never ran leaves its variable undefined.
- **A `switch` cannot be nested inside another `switch`'s `<item>` or `<default>`** — even wrapped in an `object` or `catch`. The reference parser rejects the nested `<item>` tags. Compute the inner result with `compare`/gates first, or restructure so the switches are siblings.

## See also
- [`../reference/operators-and-logic.md`](../reference/operators-and-logic.md) — when to choose `switch` vs. stacked compares, and how to handle ranges.
- [`../reference/patterns.md`](../reference/patterns.md) — multi-way mapping with a fallthrough default.
