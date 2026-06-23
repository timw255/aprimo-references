# Increment reference (`ref:increment`)

Increments a named counter and returns its new value — useful for generating a readable sequential ID instead of the default GUID.

## Syntax
```xml
<ref:increment counter="myCounter"/>
<ref:increment counter="myCounter" initialValue="125"/>
```

## Attributes
| Attribute | Required | Description |
|---|---|---|
| `counter` | recommended | Name of the counter to use or create. (See gotchas — omitting it does not fail `lint`.) |
| `initialValue` | no | Starting value used **only when the counter is first created**. Defaults to `1`. |

## Examples

Create or use a counter, seeding a new one at 125:
```xml
<ref:increment counter="myCounter" initialValue="125"/>
```
Produces: `"125"`

A brand-new counter with no `initialValue` starts at 1:
```xml
<ref:increment counter="orderNo"/>
```
Produces: `"1"`

Capture the new value silently, then build an ID with it:
```xml
<ref:text out="INV-"/><ref:increment counter="invoice" initialValue="1000"/>
```
Produces: `"INV-1000"`

## Gotchas
- `initialValue` only applies the **first** time a counter is created. If the counter already exists, it is incremented and the new value is returned; `initialValue` is ignored.
- Omitting `counter` does **not** fail `lint`; at execution it falls back to a default counter (returns `1` for a fresh one). Always name your counter explicitly.
- Counters persist in Aprimo across triggers. In this transpiler/execute harness each run is independent, so a fresh counter always returns its initial value (`1`, or `initialValue`).
- Incrementing locks a database row. Avoid high-frequency triggers on a large scope (e.g. on every record view). Up to ~20 counters per customer is supported before performance is affected.
- Migrating from a discontinued web reference for unique IDs: find the latest external counter value, add headroom for in-flight records, and pass it as `initialValue` when you switch to `ref:increment`.

## See also
- `random.md` — non-sequential numbers.
- `../reference/patterns.md` — building IDs from static text plus a reference.
