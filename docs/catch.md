# Catch reference (`ref:catch`)

Runs its child references and, if **any** of them throws, swallows the error and emits a fallback value instead. This is the crash-safety wrapper for rules that must never fail — validation routines, default values, and anything reading a field or file that might not exist. It is a **container tag**.

## Syntax
```xml
<ref:catch ex="System.Exception" out="fallback">
    ...references that might throw...
</ref:catch>
```

## Attributes
| Attribute | Description |
|---|---|
| `ex` | The exception type to catch. `System.Exception` catches **everything**; any other value catches **only** an exception of that exact .NET type (see Gotchas). The attribute is **`ex`**, not `type`. |
| `out` | The fallback value emitted when an error is caught. May reference the error variables below. |

Inside `out`, these variables are available when an error is caught:

| Variable | Resolves to |
|---|---|
| `@adamError` | The complete exception (type, message, and stack trace). |
| `@adamErrorMessage` | Just the message, e.g. `Variable '@x' does not exist.`. |
| `@adamErrorType` | The .NET exception type, e.g. `ReferenceException` or `ArgumentException`. |

## Examples

**Return a safe fallback on any error.** If the inner reference throws, `out` is emitted; otherwise the inner result passes through:
```xml
<ref:catch ex="System.Exception" out="Error">
    <ref:user name="Unknown" out="name"/>
</ref:catch>
```
Produces: `"Error"` (no user named "Unknown" exists, so the lookup throws and the fallback wins)

**Pass through when nothing fails.** With a valid lookup the inner value is returned unchanged:
```xml
<ref:catch ex="System.Exception" out="fallback">
    <ref:user out="name"/>
</ref:catch>
```
Produces: `"Test User"` (default context — the inner reference succeeds)

**Surface the error message** by putting `@adamErrorMessage` in `out` — useful while debugging a rule:
```xml
<ref:catch ex="System.Exception" out="@adamErrorMessage">
    <ref:user name="Unknown" out="name"/>
</ref:catch>
```
Produces: `"user not found with name: Unknown"`

**Guard a computation that can throw.** Reads of an undefined variable, a non-numeric `ref:eval`, and similar operations throw; wrap the rule so it always yields a safe default:
```xml
<ref:catch ex="System.Exception" out="0">
    <ref:eval expression="@count + 1"/>
</ref:catch>
```
Produces: `"0"` when `@count` was never stored (the `eval` throws and the fallback wins); otherwise the computed number.

> Reading a **missing field** does *not* throw — `<ref:record fieldName="Missing" out="value"/>` renders empty. But the value it produces is **null**, not `""`, and passing that null on to `ref:switch`, `ref:replace`, `ref:regex`, `ref:count` or `ref:datediff` *does* throw. See [Missing values are null](overview.md#missing-values-are-null).

## Gotchas
- **A `store` on an element that threw never happened.** The catch swallows the
  error, but the variable the failed element was supposed to set does not exist,
  so the next read of it fails with `Variable '@x' does not exist.` - and *that*
  error is outside the catch. If a store inside a catch matters afterwards,
  initialize it **before** the catch:

  ```xml
  <ref:text out="0" store="@cn"/>
  <ref:catch ex="System.Exception" out=""><ref:count in="@maybe" store="@cn"/></ref:catch>
  ```

  This is the usual reason a catch appears to "lose" a store.


Besides `ex` and `out`, `ref:catch` also accepts `store=` - verified live, whether or not the catch fires. As everywhere else, `store=` suppresses the element's own emission. The common formatting attributes (`left`, `right`, `case`, ...) are accepted but have NO effect on a catch.
- **`System.Exception` catches everything; a narrower `ex` matches only that exact type.** `ex="System.ArgumentException"` catches an `ArgumentException` but lets a `ReferenceException` propagate; a mismatched or base type (e.g. `System.SystemException`) does **not** catch. Use `ex="System.Exception"` for a never-fail rule.
- **Wrap the whole rule, not just one read.** For a rule that must always return a value, put all the logic inside one `catch` with the safe default as `out` (e.g. `out="false"`). Any exception anywhere inside resolves to that value.
- **`out` is only emitted on error.** When the children succeed, their output is returned and `out` is ignored — so don't use `out` as an "else" branch for non-error logic; use gated `ref:text` for that.

## See also
- [`../reference/patterns.md`](../reference/patterns.md) — "Crash-safety: wrap in `catch`" and conditional-default patterns.
- [`../reference/operators-and-logic.md`](../reference/operators-and-logic.md) — building the never-fail boolean logic you wrap in `catch`.
