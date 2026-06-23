# Date / Time (`ref:date`, `ref:now`)

Return the current date (`ref:date`) or the current date **and** time (`ref:now`). These are the starting point for relative-date math with `ref:datetimeeval`. Both produce a typed datetime VALUE — feed it straight into `datetimeeval`/`datediff` (a stored value stays typed; a value re-emitted through `ref:text` becomes a plain string).

## Syntax
```xml
<ref:date/>
<ref:now/>
```

## Attributes
| Attribute | Description |
|---|---|
| store | (optional) Variable to capture the value. With `store`, nothing is emitted — feed it into `ref:datetimeeval` or `ref:datediff`. |
| format | (optional) A .NET format string controlling the output (e.g. `yyyy-MM-dd`, `MMM d, yyyy`). Without it, the default culture format is used. |

(`ref:datetime` is accepted as an alias for `ref:now`.)

## Examples

**Current date and time** (default culture format `M/d/yyyy h:mm:ss tt`):
```xml
<ref:now/>
```
Produces: `"6/15/2026 1:57:34 PM"`

**Today** — `ref:date` is the current day at midnight:
```xml
<ref:date/>
```
Produces: `"6/15/2026 12:00:00 AM"`

**Formatted** — an ISO date string:
```xml
<ref:now format="yyyy-MM-dd"/>
```
Produces: `"2026-06-15"`

**Store now and compute a future date** — the canonical "now + N" idiom:
```xml
<ref:now store="@now"/>
<ref:datetimeeval in="@now" addYears="5" store="@future"/>
<ref:text out="@future"/>
```
Produces (relative to the run): `"6/15/2031 1:57:34 PM"`

**Store today and measure elapsed days** to a field date:
```xml
<ref:date store="@today"/>
<ref:record fieldName="ExpirationDate" out="value" store="@exp"/>
<ref:datediff date1="@today" date2="@exp" timeunit="Days" store="@daysLeft"/>
<ref:text out="@daysLeft"/>
```
Produces a (possibly fractional) day count; negative if the date has already passed.

## Gotchas
- **The default output is the culture format** `M/d/yyyy h:mm:ss tt` (US: month/day, 12-hour, AM/PM, no timezone). `ref:date` renders at midnight (`12:00:00 AM`). Use `format="..."` for any other shape.
- **Both produce a typed value.** Pass the stored `@var` straight into `datetimeeval`/`datediff`. Don't round-trip it through `ref:text` first — that yields a string, which `datetimeeval` rejects.
- **`store` suppresses output.** Use it when you only want to feed the value forward; otherwise the bare tag emits.
- **Values are evaluated at run time**, so the output reflects the moment of execution.

See also: [datetimeeval.md](datetimeeval.md) and [datediff.md](datediff.md).
