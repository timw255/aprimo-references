# DateTimeEval (`ref:datetimeeval`)

Adds or subtracts time from a datetime **value** and returns the result. This is how you compute relative dates like "now + 5 years" or "expiration − 10 days" — never write date math as text.

## Syntax
```xml
<ref:datetimeeval in="@dateValue" addDays="number"/>
```

## Attributes
| Attribute | Description |
|---|---|
| in | The input datetime. **Must be a date VALUE** — `@variable` holding the result of `ref:now`/`ref:date` or a Date/DateTime field. A literal string is rejected (see gotchas). |
| addYears | Years to add (negative subtracts). Whole number. |
| addMonths | Months to add (negative subtracts). Whole number. |
| addDays | Days to add (negative subtracts). May be fractional (`1.5`). |
| addHours | Hours to add (negative subtracts). May be fractional. |
| addMinutes | Minutes to add (negative subtracts). May be fractional. |
| addSeconds | Seconds to add (negative subtracts). May be fractional. |
| addMilliseconds | Milliseconds to add (negative subtracts). |
| format | (optional) A .NET format string controlling the output (e.g. `yyyy-MM-dd`, `MMM d, yyyy`). Without it, the default culture format is used. |
| timezone | (optional) Time zone of the result. |
| store | (optional) Variable to capture the result. With `store`, nothing is emitted. |

Combine multiple `addNNN` arguments in one reference (each at most once).

## Examples

**Now + 5 years** (the standard "retention end date" idiom):
```xml
<ref:now store="@now"/>
<ref:datetimeeval in="@now" addYears="5" store="@future"/>
<ref:text out="@future"/>
```
Produces (relative to the run): `"6/15/2031 1:53:40 PM"`

**Subtract from a field value** — a warning date 10 days before expiration:
```xml
<ref:record fieldName="ExpirationDate" out="value" store="@exp"/>
<ref:datetimeeval in="@exp" addDays="-10" store="@warn"/>
<ref:text out="@warn"/>
```
With a Date field `ExpirationDate = 2025-06-01`, produces `"5/22/2025"` (date-only, because the field is a Date).

**Format the result** — an ISO date string:
```xml
<ref:now store="@now"/>
<ref:datetimeeval in="@now" addMonths="18" format="yyyy-MM-dd" store="@dt"/>
<ref:text out="@dt"/>
```
Produces: `"2027-12-15"`

## Gotchas
- **`in` must be a datetime VALUE, never a literal string.** Aprimo's check is type-based, not format-based: `in="2024-01-15"` (or any literal, even one in the exact display format) is **rejected** — it throws. Feed it `@var` from `ref:now`/`ref:date` or a Date/DateTime field. (A value you stored as text — e.g. via `ref:text out="@now" store="@s"` — becomes a string and is also rejected.)
- **The default output is the culture format** `M/d/yyyy h:mm:ss tt` (e.g. `12/22/2027 11:49:47 PM`). Use `format="..."` for anything else. The output preserves date-only vs full: a Date field stays date-only (`12/31/2027`), a DateTime field keeps its time.
- **Month/year math CLAMPS to month-end.** `addMonths="1"` on Jan 31 returns Feb 28 (not Mar 3); `addYears` on Feb 29 lands on Feb 28 in a non-leap year.
- **`store` suppresses output.** With `store="@val"` nothing prints; emit it later with `ref:text out="@val"`.

See also: [datediff.md](datediff.md), [datetime.md](datetime.md), and [../reference/patterns.md](../reference/patterns.md).
