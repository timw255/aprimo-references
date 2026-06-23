# Translation (`ref:translation`)

Translates a key into the UI language of the current user, or into a specific language. The translated string is emitted in place.

## Syntax
```xml
<ref:translation studio="studio" module="module" key="key" [languageId="languageId"]/>
```

## Attributes
| Attribute | Required | Description |
|---|---|---|
| `studio` | Yes | The studio that owns the translation key. |
| `module` | Yes | The module that owns the translation key. |
| `key` | Yes | The key to translate. |
| `languageId` | No | The Id (Guid) of the chosen language. When omitted, the key is translated into the UI language of the current user. |
| `store` | No | Captures the translated value into an `@variable` without emitting it. |

> `ref:translation` has **no `out` attribute** — the result is the translated string itself. Adding `out` fails linting.

## Examples

> In the transpiler/test runtime, translation returns a deterministic mock of the form `Translated[studio:module:key]` (or `Translated[studio:module:key:languageId]`). In a live Aprimo environment this is the actual translated text for the key.

Translate a key into the current user's UI language:
```xml
<ref:translation studio="DAM" module="Common" key="welcome"/>
```
Produces: `Translated[DAM:Common:welcome]`

Translate a key into a specific language by its Id (Guid):
```xml
<ref:translation studio="DAM" module="Common" key="welcome" languageId="fr-FR-guid-003"/>
```
Produces: `Translated[DAM:Common:welcome:fr-FR-guid-003]`

Resolve the target language dynamically — look up a language Id with `ref:language`, store it, then translate into it:
```xml
<ref:language search="culture = fr-FR" out="id" store="@frId"/>
<ref:translation studio="DAM" module="Common" key="welcome" languageId="@frId"/>
```
Produces: `Translated[DAM:Common:welcome:fr-FR-guid-003]`

## Gotchas
- **No `out` attribute.** `<ref:translation ... out="x"/>` is rejected at lint time. The valid tag-specific attributes are `studio`, `module`, `key`, `languageId`.
- **`languageId` is a Guid**, not a culture code. Resolve it with `ref:language out="id"` when you only know the culture or name.
- **A `@variable` in `languageId` must be defined by a preceding `store`.**
- **`store` emits nothing**; the translated text only prints when the reference is ungated.

## See also
- `language.md` — resolving a language Id to pass as `languageId`.
- `../reference/patterns.md` — store-then-use patterns.
