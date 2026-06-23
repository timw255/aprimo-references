# Download order reference (`ref:downloadOrderInfo`)

Returns information about a **download order** — the file names involved, a human-readable label, and the link to the Downloads page. These references are almost always used inside the notification e-mail that is sent to a user when their download order is ready.

## Syntax

```xml
<ref:downloadOrderInfo out="out" />
```

A single self-closing tag. The `out` attribute selects which piece of information to return.

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `out` | String | Which property of the download order to return (see table below). |

### `out` values

| `out` value | Type | Description |
|---|---|---|
| `fileName` | String | The names of the first files in the download order. One file returns `"Filename1"`; two files return `"Filename1, Filename2"`; more than two files return `"Filename1, Filename2..."` (truncated with an ellipsis). |
| `fileNameLabel` | String | The string `"download of n file(s): "`, where `n` is the number of files ordered for download. Handy as a prefix in front of `fileName`. |
| `downloadsurl` | String | The link to the **Downloads** page where the ordered files can be downloaded. |

> Common attributes such as `format`, `encode`, `case`, `store`, and the gating attributes (`onVariable`, `onEmpty`, …) work here too. See [overview.md](overview.md).

## Examples

All examples below were executed against the compiler with its default sample context (`fileName = "document.pdf"`, a single-file order).

**The ordered file name(s):**

```xml
<ref:downloadOrderInfo out="fileName" />
```
Resolves to `document.pdf`.

**The descriptive label:**

```xml
<ref:downloadOrderInfo out="fileNameLabel" />
```
Resolves to `download of 1 file(s): `.

**The download link:**

```xml
<ref:downloadOrderInfo out="downloadsurl" />
```
Resolves to the Downloads-page URL, e.g. `https://example.com/downloads/default`.

**A complete notification line** combining the label, file names, and link:

```xml
<ref:downloadOrderInfo out="fileNameLabel" />
<ref:downloadOrderInfo out="fileName" />
— <a href="">
<ref:downloadOrderInfo out="downloadsurl" />
</a>
```
Produces text such as `download of 1 file(s): document.pdf — <a href="…">https://example.com/downloads/default</a>`.

## Gotchas

- **Context-bound.** `downloadOrderInfo` only resolves inside a download-order notification. Outside that context there is no order to read from, so the values are empty.
- **`fileName` is a summary, not a full list.** For orders of more than two files it deliberately truncates to the first two names plus an ellipsis. If you need every file name, iterate the order's targets with a maintenance reference instead — see [maintenance.md](maintenance.md).
- **`store` suppresses output.** Adding `store="@val"` captures the value into `@val` and emits nothing. Only an ungated reference prints. See [overview.md](overview.md).
