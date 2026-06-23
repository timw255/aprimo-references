# Random reference (`ref:random`)

Returns a random integer between `minimum` and `maximum`, inclusive of both bounds.

## Syntax
```xml
<ref:random minimum="value" maximum="value"/>
```

## Attributes
| Attribute | Required | Description |
|---|---|---|
| `minimum` | yes (effectively) | Lower bound, inclusive. Defaults to `0` if omitted. |
| `maximum` | yes (effectively) | Upper bound, inclusive. Defaults to `0` if omitted. |

Both attributes are needed for a useful range. With fewer than two numeric bounds the result collapses (see gotchas).

## Examples

> The output is random — values below are representative of one run; re-running yields different numbers in the same range.

A dice roll between 1 and 6:
```xml
<ref:random minimum="1" maximum="6"/>
```
Produces: `"5"` (a random integer in 1–6)

Capture the value silently, then emit it:
```xml
<ref:random minimum="1" maximum="6" store="@roll"/><ref:text out="@roll"/>
```
Produces: `"5"` (the captured roll)

Build a pseudo-random SKU suffix:
```xml
<ref:text out="SKU-"/><ref:random minimum="1000" maximum="9999"/>
```
Produces: `"SKU-5058"` (a random 4-digit suffix)

## Gotchas
- The range is **inclusive** on both ends: `minimum="1" maximum="6"` can return `1`, `6`, or anything between.
- Output is **non-deterministic** — each execution produces a different value. Do not write tests that assert an exact number.
- **Both bounds matter.** Internally a missing bound defaults to `0`: `maximum="3"` alone yields `0`–`3`; `minimum="5"` alone is effectively `5`–`0`, an empty range that returns small/zero values. Providing neither returns `0`. Always supply both `minimum` and `maximum`.
- `lint` does not flag a missing bound — supplying both is your responsibility.
- `store="@var"` captures the value silently; only an ungated emitting reference (e.g. `ref:text`) prints it. See `../reference/patterns.md`.

## See also
- `increment.md` — sequential (non-random) numbers.
- `../reference/patterns.md` — store/emit and ID-building patterns.
