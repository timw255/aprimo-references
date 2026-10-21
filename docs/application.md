# Application reference (`ref:application`)

Returns a property of the running Aprimo application. In practice it has one job: giving you the
tenant's **DAM base URL** so a reference can build a link to a record, an order, or a page - the
platform's own notification e-mail template uses it. It is a **leaf tag**.

## Syntax
```xml
<ref:application out="damUrl"/>
```

## Attributes
| Attribute | Required | Description |
|---|---|---|
| `out` | yes | The property to return. The only value that resolves is `damUrl` (case-insensitive). |
| `store` | no | Capture the value into an `@variable` without emitting it. |

### `out` values
| Value | Description |
|---|---|
| `damUrl` | The DAM base URL of the tenant, e.g. `https://acme.dam.aprimo.com/dam`. |

Any other value throws `Attribute 'out' has an invalid value '<value>' for node 'application'.`
(the value is reported lowercased). `out="id"` is accepted by the parser but throws a
`NullReferenceException` at run time, so treat `damUrl` as the whole API.

## Examples

Emit the base URL:
```xml
<ref:application out="damUrl"/>
```
Produces: `https://tenant.dam.aprimo.com/dam` (default context; the real tenant URL live)

Build a link to the current record for an e-mail body - the id goes in without dashes:
```xml
<ref:application out="damUrl" store="@damUrl"/>
<ref:record out="id" store="@id"/>
<ref:replace in="@id" oldValue="-" newValue="" store="@id"/>
<ref:object><a href="@damUrl/contentitems/@id"><ref:record out="title"/></a></ref:object>
```
Produces: `<a href="https://tenant.dam.aprimo.com/dam/contentitems/0784...">Quarterly Report</a>`

## Gotchas
- **`out` is required and only `damUrl` works.** There is no `moUrl`, `apiUrl`, `tenant`,
  `environment` or `version` projection (each is the invalid-value error).
- **Testing locally.** The default context carries `application.damUrl`; override it with
  `-context-json '{"application":{"damUrl":"https://acme.dam.aprimo.com/dam"}}'`.
- `@variables` substitute inside the **attributes** of raw HTML (`href="@damUrl/..."`), not in
  its text - see [object.md](object.md).

## See also
- [`record.md`](record.md) - `out="id"` and `out="title"` for the link text.
- [`../reference/automation-patterns.md`](../reference/automation-patterns.md) - the Send e-mail template.
