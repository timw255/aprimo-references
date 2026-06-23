# Emitting markup & references as text (HTML output, `result="ref"`, loops)

References output strings. Two cases need extra care because the output *itself* is
markup or another reference: **emitting HTML**, and **building a reference in a variable and
re-resolving it** with `result="ref"` (which is how you get dynamic references and loops).
Everything here is verified against the compiler.

> The HTML-emitting part is routine. The `result="ref"` loop part is **advanced and fiddly**
> (multi-level escaping, easy to get wrong) — reach for it only when there's no ordinary way to
> do the job: prefer `ref:foreach` whenever you have a collection to iterate. But when you need
> bounded iteration and have *no* driver collection, it's a real unlock — so it's documented
> here. Always `execute` it to confirm.

## Emitting literal HTML

`out=` is an XML attribute, so literal `<`, `>`, `"`, `&` must be **XML-entity-encoded**; the
engine decodes them and the field receives real HTML.

```xml
<ref:record fieldName="SKU" out="value" store="@sku"/>
<ref:text out="&lt;img src=&quot;https://barcodeapi.org/api/code128/@sku&quot; alt=&quot;@sku&quot;/&gt;"/>
```
Output: `<img src="https://barcodeapi.org/api/code128/ABC-12345" alt="ABC-12345"/>` — a working
barcode image. `@sku` interpolates normally inside the encoded HTML (it ends at the next `/`,
`"`, or space).

- **Gotcha — CSS at-rules collide with the `@` sigil.** `<style>@import …` throws at run time
  (`Variable '@import' does not exist.`), because `@import` is read as a variable. Load
  fonts/styles with `<link rel="stylesheet" href="…">` (no `@`), or write a literal `@` as
  `&#64;`.
- Need the value HTML-safe (it may contain `<`/`&`)? Read it with `encode="html"`.

## Re-resolving text as a reference (`result="ref"`)

`result="ref"` re-parses a value's **text** as a new reference and runs it, sharing the same
`@variables` (one global dictionary). It's the only "evaluate built-up reference code"
mechanism — the basis for dynamic and looping references.

The trick is **deferring** the inner `@variables` and tags so they aren't consumed at the
current parse, only when the text is re-resolved:

- **Defer a `@variable` one level:** write `@` as `&#64;`. In a *source* attribute (which is
  itself parsed once), write `&amp;#64;` — it becomes `&#64;` after the first parse, then `@`
  when re-resolved.
- **Inner tags:** `&lt;ref:… /&gt;` become real `<ref:…>` after the first parse, then execute
  on re-resolution.
- **Each extra `result="ref"` nesting = one more level of escaping.**

## Loops via re-resolution

**First choice is always `ref:foreach`** when you have a collection. The patterns below are the
escape hatch for when you don't — bounded iteration with no driver collection. References have
**no `while`**; both are bounded (you fix the max up front).

### A) Recursion — capped at 3 (avoid for real loops)

A body that re-emits itself with `result="ref"` and a decremented counter recurses, but
`result="ref"` is **hard-capped at 3 nested levels** (`result='ref': recursion limit of 3
exceeded`) — and the terminating gate costs one extra level to check, so it errors at a counter
of 3 already. Usable only for a fixed depth of ~2, not for counting loops.

### B) Geometric unroll — the scalable pattern

Build the one-iteration body once, concatenate it **geometrically** into a big template, and
fire it with a **single** `result="ref"`. Each copy is gated on a counter and decrements it, so
the first N copies run and the rest no-op. One re-resolution level → the recursion cap never
trips.

```xml
<!-- one iteration: emit @Count and a space, then decrement (@ deferred as &amp;#64;) -->
<ref:text store="@Tmpl" out="&lt;ref:object onVariable='IsNotZero(&amp;#64;Count)'&gt;&lt;ref:text out='&amp;#64;Count'/&gt;&lt;ref:text out=' '/&gt;&lt;ref:eval expression='&amp;#64;Count-1' store='&amp;#64;Count'/&gt;&lt;/ref:object&gt;" />

<!-- expand x5 each step: 5 -> 25 -> 125 sequential copies -->
<ref:text store="@Tmpl5"   out="@Tmpl @Tmpl @Tmpl @Tmpl @Tmpl" />
<ref:text store="@Tmpl25"  out="@Tmpl5 @Tmpl5 @Tmpl5 @Tmpl5 @Tmpl5" />
<ref:text store="@Tmpl125" out="@Tmpl25 @Tmpl25 @Tmpl25 @Tmpl25 @Tmpl25" />

<!-- set the counter and run all 125 copies once -->
<ref:text store="@Count" out="75" />
<ref:text result="ref" out="@Tmpl125" />
```
Output: `75 74 73 … 2 1 ` — the first 75 copies emit and decrement; the remaining 50 see
`@Count = 0`, fail their gate, and emit nothing. (Verified.)

Copies grow geometrically (5 expansion stores → 5·25·125·625·3125…), so thousands of
iterations cost a handful of lines. **The loop ceiling is the unroll size, not 3.** Set the
number of copies ≥ your maximum counter.

> This does **not** make references Turing complete — the iteration count is still chosen before
> the run (by how many copies you emit). But when you need "loop up to N" and have no driver
> collection to `foreach`, unroll a template to ≥N copies and gate each on a counter.

## See also
- `computational-model.md` — the construct index, including loops.
- `httprequest.md` — when a value must be *computed* (not just assembled), call out instead.
