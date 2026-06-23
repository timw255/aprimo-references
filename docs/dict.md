# Dictionary reference (`ref:dict`)

Inserts a translatable dictionary term into a field's value. The stored value is the reference itself; it is resolved into translated text for every field-enabled language.

## Syntax
```xml
<ref:dict name="termKey"/>
<ref:dict key="termKey"/>
```

## Attributes
| Attribute | Required | Description |
|---|---|---|
| `name` | one of `name`/`key` | The dictionary term name (key) to resolve. |
| `key` | one of `name`/`key` | Alias for `name`; either works. |

## Examples

Resolve a dictionary term by its key:
```xml
<ref:dict name="welcome"/>
```
Produces: `"Welcome"`

A term whose value contains punctuation/words:
```xml
<ref:dict name="hello_user"/>
```
Produces: `"Hello, User!"`

Another term:
```xml
<ref:dict name="thank_you"/>
```
Produces: `"Thank you"`

`key` is an accepted alias for `name`:
```xml
<ref:dict key="welcome"/>
```
Produces: `"Welcome"`

Combine static text with a translatable term:
```xml
<ref:text out="12 bottles - "/><ref:dict name="thank_you"/>
```
Produces: `"12 bottles - Thank you"`

## Gotchas
- A term key that does not exist resolves to an **empty string** — no error, no fallback to the key name.
- Keys are matched exactly and are case-sensitive (`welcome`, not `Welcome`).
- In real Aprimo the resolved value depends on the field's language; the same reference yields the term's translation in each field-enabled language, and all translations are written to the full-text index so users can search by any of them.
- To use dictionary references in fields you must: (1) create a dictionary term (managed as a translation in System — Studio `Dictionary`, Module `Default`), (2) create a Text field with the References property enabled (Multi Line recommended), and (3) enter the dictionary reference as the field value.

## See also
- `translation.md` — single-language translation lookups.
- `../reference/patterns.md` — combining references and static text.
