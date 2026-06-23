# DateDiff (`ref:datediff`)

Calculates the elapsed time between two dates/datetimes in a chosen unit. Positive when `date1` is earlier than `date2`, negative when `date1` is later. The result is **fractional** (a real number), not rounded.

## Syntax
```xml
<ref:datediff date1="someDate" date2="otherDate" timeunit="Days" store="@diff"/>
```

## Attributes
| Attribute | Description |
|---|---|
| date1 | The first date/datetime. A date VALUE (`@var`) or a literal string — unlike `datetimeeval`, `datediff` accepts literal date strings (ISO `2026-01-31` or US `1/31/2026`). |
| date2 | The second date/datetime (same rules as `date1`). |
| timeunit | Unit to measure in (case-insensitive). Defaults to `Days`. Valid: `Days`, `Months`, `Years`, `Hours`, `Minutes`, `Seconds`. Any other unit is an error. |
| store | (optional) Variable to capture the result. With `store`, nothing is emitted. |

## Examples

**Days between two dates:**
```xml
<ref:datediff date1="2024-01-01" date2="2024-01-15" timeunit="Days" store="@days"/>
<ref:text out="@days"/>
```
Produces: `"14"`

**Fractional days** when a time-of-day is involved:
```xml
<ref:datediff date1="2024-01-01T00:00:00Z" date2="2024-01-02T18:00:00Z" timeunit="Days" store="@days"/>
```
Produces: `"1.75"`

**Field-to-field** — days between an approval and a publication field:
```xml
<ref:record fieldName="ApprovalDate" out="value" store="@appr"/>
<ref:record fieldName="PublicationDate" out="value" store="@pub"/>
<ref:datediff date1="@appr" date2="@pub" timeunit="Days" store="@diff"/>
<ref:text out="@diff"/>
```
Produces: `"30"` (with `ApprovalDate = 2024-01-01`, `PublicationDate = 2024-01-31`).

## Gotchas
- **Every unit is fractional.** `Days`/`Hours`/`Minutes`/`Seconds` are the exact elapsed TimeSpan. **`Years` = total days / 365** (so 546 days → `1.4958…`, *not* calendar years). **`Months`** is calendar-aware: whole months elapsed plus a fractional remainder (e.g. `2026-01-15` → `2026-03-10` is `1.821…`). If you want a whole number, round it yourself (e.g. compare against thresholds).
- **Sign matters.** A negative result means `date1` is *after* `date2`. To test "is the expiration date in the past," put today as `date1` (or check the sign).
- **`datediff` is lenient about input** (accepts literal date strings in ISO or US format, and date values), unlike `datetimeeval` which requires a typed value.
- **For simple before/after, you don't need `datediff`** — `ref:compare` works directly on dates (`compare value1="@exp" value2="2026-12-31" operator="gt"`). Use `datediff` only when you need the *amount* of time.
- **`store` suppresses output.** Without `store`, the diff emits directly; with it, emit later via `ref:text`.

See also: [datetimeeval.md](datetimeeval.md), [compare.md](compare.md), and [../reference/patterns.md](../reference/patterns.md).
