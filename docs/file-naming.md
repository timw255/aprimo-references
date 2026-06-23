# File naming conventions

> Content > DAM Administration > Configuration areas > File Naming Conventions

File names matter — for clarity, consistency, and downstream usage. When files are ordered, downloaded, or bundled into a delivery, Aprimo lets you define flexible naming rules so files are clearly labeled and free of conflicts. A naming convention is a **reference template** (the same `ref:` syntax used everywhere else) plus a set of **context-injected variables** that the order/rendition agent supplies at naming time.

This page covers:

- [Where naming conventions are configured](#settings-that-define-naming-conventions)
- [The references and variables you build them from](#references--variables)
- [Worked examples](#examples), from simple to advanced

> **New conventions are not retroactive.** Updating a convention does **not** rename existing files or download orders. It applies only when a new order, rendition, or transcription is created or re-created.

## Settings that define naming conventions

### Ordered files

The ordering agent can order many files at once and combine high-res files, previews, additional files, and thumbnails into a single order. That can produce two files with the same name, conflicting at the delivery location (file system or zip). These settings control how each type of ordered file is named:

| Setting | Label | Default behavior (example) |
|---|---|---|
| `.orderDocumentNamingConvention` | Naming convention used when ordering documents | basename + attempt number + extension. Ordering `test.jpg` (2nd attempt) ⇒ `test#02.jpg`. |
| `.orderResizeNamingConvention` | Naming convention when resizing ordered files | basename + original extension + attempt + resized extension. Ordering `test.tif` (2nd attempt) as jpg ⇒ `test_tif#02.jpg`. |
| `.orderPreviewNamingConvention` | Naming convention used when ordering previews | basename + original extension + `Prev` + preview index + attempt + preview extension. Preview of `test.tif` (1st preview, 2nd attempt) ⇒ `test_tif_Prev_0#02.jpg`. |
| `.orderThumbnailNamingConvention` | Naming convention used when ordering thumbnails | basename + original extension + `Thumb` + preview index + attempt + preview extension. ⇒ `test_tif_Thumb_0#01.jpg`. |
| `.orderAdditionalFileNamingConvention` | Naming convention used when ordering additional files | basename + attempt + extension. Ordering `test.xml` (2nd attempt) ⇒ `test#02.xml`. |
| `.orderZipNamingConvention` | Naming convention for zip-files | maintenance-job ID + `zip` extension. ⇒ `50735bfd-398f-4b7d-b3b8-a2ec009e20c1.zip`. |
| `.orderPublicLinkNamingConvention` | Naming convention for public links | a folder named with the file-version ID, plus a file name concatenating file name and rendition name. |

> **Downloading does not rename in DAM.** When a file is downloaded, only the *ordered* file name changes; the file's name inside DAM is unchanged.

> If the rendition name is included in the public-link convention, the result is also influenced by the **rendition** naming convention (below).

### Generated files (renditions & transcriptions)

These settings define a uniform filename pattern for every generated rendition or transcription — created via rules (e.g. on upload) or a custom download action. Updates take effect immediately for any new or re-created rendition/transcription, both in the UI and on download.

| Setting | Label | Default behavior (example) |
|---|---|---|
| `.renditionNamingConvention` | Naming convention used when creating renditions | basename + `_rendition_` + preset name + extension. A square PNG of `test.jpg` ⇒ `test_rendition_Square.png`. |
| `.videoTranscriptFileNamingConvention` | Naming convention for transcriptions of videos | basename + language name + transcription extension. A French transcript of `test.mp4` ⇒ `test_Francais (Francais).vtt`. |

> After updating a generated-file convention, newly downloaded renditions may not match existing ones. Run **(Re)create renditions** to bring existing rendition filenames into line.

> Ordered renditions with the same name are suffixed with `#0[attempt]` to stay unique, just like the `@ATTEMPT` variable does for ordered files.

> Changing `.renditionNamingConvention` does **not** retroactively change existing public-link names that use `@RENDITIONNAME`, even after recreating the links — so links already embedded on websites keep working.

## References & variables

A naming convention is a **reference template**: you write `ref:` tags (most commonly [`ref:text`](text.md)) and weave in the variables below. Because these are ordinary references, you can also call [`ref:record`](record.md), [`ref:classification`](classification.md), [`ref:foreach`](foreach.md), etc. to pull in field values — see Examples 6 and 7. For shared attributes (alignment, gating, `store`) see [overview.md](overview.md).

### Variables

These variables are **injected by the order/rendition agent** while it names a file — they are *not* general global variables. In a naming template you can reference them directly (and the compiler accepts them syntactically), but they only carry real values inside an actual ordering/rendition context.

| Variable | Description |
|---|---|
| `adamRecord` | The record ID, if available. |
| `adamFile` | The file ID, if available. |
| `FILENAME` | The complete file name of the ordered file, e.g. `test.jpg`. |
| `BASENAME` | The basename of the ordered file, e.g. `test` for `test.jpg`. |
| `EXTENSION` | The extension of the ordered file, e.g. `jpg`. |
| `FILEVERSIONFILENAME` | The file name of the ordered file version, e.g. `test.jpg`. |
| `FILEVERSIONBASENAME` | The basename of the ordered file version, e.g. `test`. |
| `FILEVERSIONEXTENSION` | The extension of the ordered file version, e.g. `jpg`. Empty if the file has no extension. |
| `FILEVERSIONAUTOEXTENSION` | The file's extension if it has one, otherwise the extension of its file type. Use this to guarantee an extension. |
| `ADDITIONALFILE` | The ordered additional file. Read its properties via references, e.g. `<ref:record additionalfile="@ADDITIONALFILE" out="id"/>`. |
| `FILEVERSIONNUMBER` | The version number of the ordered file version, e.g. `2`. |
| `INDEX` | The index of the preview/thumbnail/additional file within the ordered version. |
| `ATTEMPT` | Avoids duplicate names. First resolved **undefined**; if the name already exists, retried with `1`, then `2`, … See [the @ATTEMPT pattern](#the-attempt-pattern). |
| `FILEVERSIONID` | (public links only) The ordered file version's ID. |
| `RENDITIONNAME` | (public links only) The rendition name of the public link being ordered. |

> The source documentation also includes an availability matrix showing which variables apply to which setting (thumbnails, previews, documents, additional files, resized documents, generated renditions, generated transcriptions). Refer to the source for the full matrix.

### Two rules that trip people up

1. **`_` ends a variable name.** Variable names are alphanumeric only, so `@BASENAME_Prev` is read as the variable `@BASENAME` followed by the literal text `_Prev`. This is exactly why the templates below interleave variables and literal separators the way they do — it works *because* the underscore is not part of the variable. (Compiler-verified.)
2. **Newlines are ignored in these settings.** Although whitespace is normally significant in references, you may add newlines inside a naming-convention setting to make the template readable; they are stripped.

### The `@ATTEMPT` pattern

`ATTEMPT` is special. The agent first resolves the name with `ATTEMPT` *undefined*; only if that name already exists in the order does it define `ATTEMPT` as `1`, then `2`, and so on, re-resolving each time. The idiom is to gate a separator on whether `ATTEMPT` (or a value derived from it) exists, so the first, unique file gets a clean name and only duplicates get a suffix — see Examples 4 and 5.

## Examples

The following examples define a name for ordered previews, stored in `.orderPreviewNamingConvention`, from simple to advanced.

### Example 1 — basic

```xml
<ref:text out="@FILEVERSIONBASENAME_Prev.@EXTENSION"/>
```
A file `test.tif` with a JPG preview becomes `test_Prev.jpg`.

Limitations:

- The name doesn't say the original was a TIF (vs. EPS, etc.).
- Ordering two `test.tif` files fails — both resolve to the same name.

### Example 2 — disambiguate by source extension + attempt

```xml
<ref:text out="@FILEVERSIONBASENAME_@FILEVERSIONEXTENSION_Prev@ATTEMPT.@EXTENSION"/>
```
Ordering the same previews yields `test_tif_Prev.jpg` and `test_tif_Prev1.jpg`.

Drawback: for a multi-page document (e.g. `test.qxd`), all page thumbnails land in one order and you lose page sort-order.

### Example 3 — add the preview index

```xml
<ref:text out="@FILEVERSIONBASENAME_@FILEVERSIONEXTENSION_Prev_@INDEX_@ATTEMPT.@EXTENSION"/>
```
Ordering two `test.tif` files yields `test_tif_Prev_1_.jpg` and `test_tif_Prev_1_1.jpg`.

Drawback: the first file's name ends in a trailing underscore (`…_1_.jpg`) — not clean.

### Example 4 — gate the suffix on `@ATTEMPT`

```xml
<ref:text out="#@ATTEMPT" store="@INDEX" onAnyVariable="@INDEX"/>
<ref:text out="@FILEVERSIONBASENAME_@FILEVERSIONEXTENSION_Prev_@INDEX@ATTEMPT.@EXTENSION"/>
```
Here `@INDEX` is only defined when the name already exists in the order: the first `ref:text` is gated by `onAnyVariable="@INDEX"`, so on the first (unique) use it does nothing. On a duplicate, `ATTEMPT` is `1`, the gated text resolves to `#1`, and the two previews become `test_tif_Prev_1.jpg` and `test_tif_Prev_1#1.jpg`.

### Example 5 — force two-digit suffixes

The index maxes out at 99, so you may want two digits:

```xml
<ref:text out="@ATTEMPT" right="2" padding="0" store="@INDEX" onAnyVariable="@INDEX"/>
<ref:text out="#@ATTEMPT" store="@INDEX" onAnyVariable="@INDEX"/>
<ref:text out="@FILEVERSIONBASENAME_@FILEVERSIONEXTENSION_Prev_@INDEX@ATTEMPT.@EXTENSION"/>
```
The first line right-aligns `ATTEMPT` with a leading `0`, but only when it's defined. The two names become `test_tif_Prev_1.jpg` and `test_tif_Prev#01.jpg`.

The point to remember: adding `@ATTEMPT` to the convention is what tells Aprimo how to rename on a duplicate.

> For `.orderDocumentNamingConvention`, prefer `@FILEVERSIONAUTOEXTENSION` over `@FILEVERSIONEXTENSION`. `FILEVERSIONEXTENSION` returns an empty string when the ordered file has no extension; `FILEVERSIONAUTOEXTENSION` falls back to the file type's extension, so ordered files always have a usable extension (so PC users can open them).

### Example 6 — readable public links from field values

You can pull field data into a public-link convention — e.g. campaign name (a classification) and brand (a text field):

```xml
<ref:record fieldName="MyCampaignClassificationListField" out="value" store="@classificationId" />
<ref:classification id="@classificationId" out="name" />
<ref:text out="/" />
<ref:record fieldName="MyBrandTextField" out="value" />
<ref:text out="/@FILEVERSIONBASENAME" />
<ref:text out="_@RENDITIONNAME.@EXTENSION" />
```
Produces a path such as `https://p1.aprimocdn.net/myenvironment/Summer/Headphones/originalfilename_rendition.jpg`. (Lint-verified.)

### Example 7 — public links for additional files, including usage

```xml
<ref:text out="@FILEVERSIONID" />
<ref:text out="/" />
<ref:object onVariable="IsNotNull(@ADDITIONALFILE)">
    <ref:record additionalFile="@ADDITIONALFILE" out="usages" store="@usages" />
    <ref:count in="@usages" store="@usagesCount"/>
    <ref:foreach in="@usages" storeitem="@usage" join="_" onVariable="IsNotZero(@usagesCount)">
        <ref:text out="@usage"/>
    </ref:foreach>
    <ref:text out="/" onVariable="IsNotZero(@usagesCount)"/>
</ref:object>
<ref:text out="@FILEVERSIONBASENAME" />
<ref:text out="_@RENDITIONNAME"/>
<ref:text out=".@EXTENSION"/>
```
Produces a URL such as `https://p1.aprimocdn.net/myEnvironment/fileversionid/usage1_usage2/originalfilename_rendition.jpg`. The `ref:object` block only contributes the usage segment when there *is* an additional file with usages.

## See also

- [overview.md](overview.md) — variables, gating, alignment, and the rules these templates rely on
- [text.md](text.md), [record.md](record.md), [classification.md](classification.md), [foreach.md](foreach.md), [object.md](object.md), [count.md](count.md) — the tags used above
- [`../reference/patterns.md`](../reference/patterns.md) — more verified idioms
