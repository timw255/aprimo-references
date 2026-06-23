# Replace (`ref:replace`)

Replaces every occurrence of one piece of text with another. The classic use is stripping whitespace out of a value before a numeric compare.

## Syntax
```xml
<ref:replace in="value" oldValue="old" newValue="new"/>
```

## Attributes
| Attribute | Description |
|---|---|
| in | The text to operate on (literal or `@variable`). |
| oldValue | The substring to find. **Case sensitive.** |
| newValue | The replacement. Use `""` to delete `oldValue`. |
| store | (optional) Variable to capture the result. With `store`, nothing is emitted. |

## Examples

**Basic substitution:**
```xml
<ref:replace in="hello world" oldValue="world" newValue="there"/>
```
Produces: `"hello there"`

**Replaces all occurrences** (not just the first):
```xml
<ref:replace in="a-b-c-d" oldValue="-" newValue="_"/>
```
Produces: `"a_b_c_d"`

**Trim whitespace before a numeric compare** — a padded value silently fails a numeric compare, so strip spaces first:
```xml
<ref:replace in=" 100 " oldValue=" " newValue="" store="@clean"/>
<ref:compare value1="@clean" value2="50" operator="gt" store="@ok"/>
<ref:text out="@ok"/>
```
Produces: `"True"` (without the trim, `compare value1=" 100 " ...` returns `"False"`)

**Case sensitivity** — only the exact-case match is replaced:
```xml
<ref:replace in="Hello hello" oldValue="hello" newValue="X"/>
```
Produces: `"Hello X"`

## Gotchas
- **Case sensitive.** `oldValue="hello"` does not match `Hello`. To normalize case-insensitively you would need to know the exact casing in advance.
- **`store` suppresses output.** With `store="@val"` the replace prints nothing; chain it into a later compare or text.
- **It is a plain string replace, not regex.** `oldValue` is matched literally. For pattern-based extraction or matching, use [regex.md](regex.md).
- **`oldValue` cannot be empty.** An empty `oldValue` is an error. Provide the exact substring to find.
- **Whitespace-trim idiom:** `oldValue=" " newValue=""` removes all spaces. This is the standard fix when a field value comes in padded and breaks a numeric `ref:compare`.

See also: [../reference/patterns.md](../reference/patterns.md).
