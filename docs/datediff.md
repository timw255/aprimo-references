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
| `Years` | Whole anniversaries plus the leftover days over the number of days in the **calendar year of `date2`** (365, or 366 in a leap year). `2023-06-15` to `2023-08-27` is 73/365 = `0.2`; the same 73-day span in 2024 is 73/366 = `0.1994535519125683`. Live-fitted over a 23-case leap-boundary sweep (wave-103). |
- **Sign matters.** A negative result means `date1` is *after* `date2`. To test "is the expiration date in the past," put today as `date1` (or check the sign).
- **`datediff` is lenient about input** (accepts literal date strings in ISO or US format, and date values), unlike `datetimeeval` which requires a typed value.
- **For simple before/after, you don't need `datediff`** — `ref:compare` works directly on dates (`compare value1="@exp" value2="2026-12-31" operator="gt"`). Use `datediff` only when you need the *amount* of time.
- **`store` suppresses output.** Without `store`, the diff emits directly; with it, emit later via `ref:text`.

See also: [datetimeeval.md](datetimeeval.md), [compare.md](compare.md), and [../reference/patterns.md](../reference/patterns.md).

## Missing dates throw

`date1`/`date2` must resolve to real dates. A field that the record does not
carry reads as NULL, and `ref:datediff` then throws an invalid-value error
rather than returning `0`. Gate the whole computation:

```xml
<ref:record fieldName="Expiry" out="value" store="@expiry"/>
<ref:object onVariable="IsNotEmpty(@expiry)">
  <ref:datediff date1="@asOf" date2="@expiry" timeunit="days"/>
</ref:object>
```
