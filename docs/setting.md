# Setting reference (`ref:setting`)

Retrieves a resolved setting value, honoring overrides (e.g. a user-level value overriding the system value).

## Syntax
```xml
<ref:setting name="settingKey"/>
<ref:setting key="settingKey"/>
```

## Attributes
| Attribute | Required | Description |
|---|---|---|
| `name` | one of `name`/`key` | The name of the setting to retrieve. |
| `key` | one of `name`/`key` | Alias for `name`; either works. |

At least one of `name` or `key` is required — omitting both fails `lint`:
`ref:setting requires 'name' or 'key' attribute - use name="settingKey"`.

## Examples

Setting names are the platform's **exact keys**, and they **differ per tenant and change over
time** - built-in keys come and go with releases, and custom definitions are whatever the admin
created. A name that does not exist throws (see Gotchas), so **never guess a key**: ask the user
for it, or have them read it from *Settings* in the admin UI (or `GET /api/core/settingdefinitions`).
The only key quoted from Aprimo's own documentation is `.technicalContactEmail`, the `<cc>` of the
stock Send-email template. There is **no** `.damUrl` setting - the DAM base URL comes from
[`ref:application out="damUrl"`](application.md).

Read the technical contact:
```xml
<ref:setting name=".technicalContactEmail"/>
```
Produces: `"dam-admin@example.com"` (default context; the tenant's own value live)

Read a custom setting by its exact name:
```xml
<ref:setting name="ExampleCustomSetting"/>
```
Produces: `"example value"` (default context)

`key` is an accepted alias for `name`:
```xml
<ref:setting key="ExampleCustomSetting"/>
```
Produces: `"example value"`

Capture silently with `store`, then emit:
```xml
<ref:setting name="ExampleCustomSetting" store="@cfg"/><ref:text out="@cfg"/>
```
Produces: `"example value"`

## Gotchas
- A setting that does not exist **throws**: `No setting found matching the specified criterion.`
  It does not resolve to an empty string, and there is no default. Wrap the
  reference in `ref:catch` if the setting may be absent — an uncaught throw aborts the whole
  evaluation, which is exactly the situation where someone forgot to create the setting.
- A **present but empty** `name=""` throws the same way; an absent `name=` is the
  mandatory-attribute error instead.
- Setting names are matched exactly, including any leading dot (e.g. `.systemName`). Use the exact key as configured.
- The returned value is the **resolved** value: if a user-level value overrides the system value, the user value is returned.
- A `store` attribute captures without emitting; pair with an ungated `ref:text` to print. See `../reference/patterns.md`.

## Custom settings

You are not limited to the built-in (dotted) settings. Aprimo admins can define **custom setting
definitions** of various data types (Boolean, Numeric, DateTime, Text, XML, and a Reference type
for reference content). `ref:setting name="YourSetting"` reads any of them the same way (custom
names are alphanumeric with no leading dot; built-in ones start with `.`).

Reach for this to **externalize** things instead of baking them into the reference — tunable
thresholds and limits, allowed-value lists, key→value mappings, environment URLs/config, even
reusable reference snippets. When a rule would otherwise carry a big `switch`, a long hardcoded
list, or environment-specific constants, that data usually belongs in a setting the reference
reads: it's editable without touching (or redeploying) the reference. Defining the setting is an
admin task in the Aprimo UI; from the reference side you just read it.

**A distribution list is a setting.** References cannot enumerate a user group's members, so
when an e-mail must go to a team, keep the addresses in a custom Text setting
(`SecurityTeamRecipients` = `a@x.com;b@x.com`) and read it in the `<recipients>` slot. See
[usergroup.md](usergroup.md).

**Testing a setting locally.** You don't need the real setting to exist — add it (built-in or
custom) to the `settings` map in your context and the compiler resolves it:

```bash
echo '<ref:setting name="MaxRetentionYears"/>' \
  | bin/transpiler.exe -mode execute -context-json '{"settings":{"MaxRetentionYears":"7"}}'
# -> "7"
```

`-mode dump-context` shows the settings already in the default context; anything you put in
`settings` overrides/extends them (see `../reference/compiler.md`).

## See also
- `../reference/patterns.md` — store/emit patterns.
