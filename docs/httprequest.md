# HTTP request reference (`ref:httpRequest`)

Calls an external service (the Aprimo REST API, an Azure Function, an AWS Lambda, a Google Cloud Function, etc.) so you can do more advanced calculations and processing than the built-in references allow. **The value returned by the HTTP response becomes the value of the reference.**

`ref:httpRequest` is a **container**: its child elements (`Request` / `Headers` / `Header` / `Body`, or a bare body) are parsed, and any references nested inside the body are resolved *before* the request is sent. The body you build can mix literal markup (JSON, XML, SOAP, YAML, CSV) with nested references.

> **Attention:** Immediate execution of reference-rule actions that contain HTTP request references has been deprecated. Change the rule action's execution mode to **delayed**.

## Syntax

There are two ways to write the request: **structured** (with explicit headers) or **body-only** (just a payload).

**Structured** — wrap headers and body in `<Request>`:

```xml
<ref:httpRequest uri="https://YourFunction.azurewebsites.net/api/..." type="application/json" timeout="0.1" retryCount="3" downTime="120">
  <Request>
    <Headers>
      <Header name="headerName1">value</Header>
      <Header name="headerName2">value</Header>
    </Headers>
    <Body>
      ...
    </Body>
  </Request>
</ref:httpRequest>
```

**Body-only** — if you omit the `<Request>` wrapper, *everything* inside the element is treated as the request body:

```xml
<ref:httpRequest uri="https://YourFunction.azurewebsites.net/api/..." type="text/plain" timeout="0.1" retryCount="3" downTime="120">
{
  "variable1":"value",
  "variable2":"value"
}
</ref:httpRequest>
```

> The structural elements are capitalized: `Request`, `Headers`, `Header`, `Body`. This casing is accepted by the compiler.

## Attributes

The `ref:httpRequest` root element supports these attributes:

| Attribute | Description |
|---|---|
| `uri` | **Required.** The URL of the service to call. |
| `type` | (optional) The MIME type of the request. Supported: `application/json`, `application/xml`, `text/plain`. If omitted, Aprimo DAM auto-detects it. |
| `timeout` | (optional) Seconds Aprimo DAM waits for a response. Default `5`, maximum `15`. |
| `retryCount` | (optional) Number of times to retry the request if the external service returns an error. |
| `downTime` | (optional) Seconds before Aprimo DAM may call the external service again after all `retryCount` retries have failed. Downtime is **not** triggered when the service responds with `400 (Bad Request)` or `500 (Internal Server Error)`. |
| `hmacHeader` | (optional) Name of the HMAC header used to prove a request genuinely came from Aprimo DAM and was not tampered with (see [HMAC](#hmac)). When set, a header of this name is added containing a SHA-256 hash of the body combined with the secret in the `.hmacSecret` setting. |

> **Attention:** The `hmacHeader` name must be unique — it cannot match any custom header you provide or a standard HTTP header (e.g. `Authorization`). We recommend `X-Authorization`.

### Headers

Provide HTTP headers inside `<Headers>` — for example an API key or an authorization header:

```xml
<ref:httpRequest uri="http://somedomain.net/api/endpointWithAuth">
  <Request>
    <Headers>
      <Header name="Authorization">Basic S29yZWliYTo3ZjNiMGMxYzYwN2M0M2VhODg3ODY3ZjY3OGI2ZDVlMQ==</Header>
    </Headers>
  </Request>
</ref:httpRequest>
```

Restrictions:

- Every `<Header>` must have a `name` attribute, and the name must be unique within the request.
- `Keep-Alive` and `Content-Type` headers are **not** supported — they would conflict with the request's own `type` and `timeout` attributes.
- For troubleshooting, include the `referenceId` header in the requests your service makes back to DAM (see [Troubleshooting](#troubleshooting)).

### Body

You can put references in the body; they are resolved and the resolved values are sent with the request. The resolved values may themselves contain further references.

```xml
<ref:httpRequest uri="https://YourFunction.azurewebsites.net/api/...">
{
  "fileName":<ref:record file="master" out="filename" encode="json" />,
  "usertoken":<ref:user out="authtoken" encode="json" />
}
</ref:httpRequest>
```

This sends the master file's name and the user's auth token to an Azure Function. **`encode="json"` wraps each value in quotes and escapes any characters that aren't valid inside a JSON string**, so the result is well-formed JSON. To pass a user's API authorization token, use `out="authtoken"` on a user reference (see [user.md](user.md)).

## Examples

> **About execution:** In `-mode execute` the compiler does **not** make a network call. Instead it resolves all nested references and **returns the fully-resolved request body** — exactly what *would* be sent. This makes it easy to verify that your body renders correctly. The examples below were run this way against the compiler's default sample context.

**Body-only, JSON with nested references** (verified):

```xml
<ref:httpRequest uri="https://example.com/api">
{
  "fileName":<ref:downloadOrderInfo out="fileName" encode="json" />,
  "label":<ref:downloadOrderInfo out="fileNameLabel" encode="json" />
}
</ref:httpRequest>
```
Resolved body returned by the compiler:
```json
{
  "fileName":"document.pdf",
  "label":"download of 1 file(s): "
}
```
Note how `encode="json"` turned each value into a quoted, escaped JSON string.

**Structured request with an auth header and a JSON body** (verified):

```xml
<ref:httpRequest uri="https://example.com/api" type="application/json">
  <Request>
    <Headers>
      <Header name="Authorization">Bearer abc123</Header>
    </Headers>
    <Body>
{ "user":<ref:user out="name" encode="json" /> }
    </Body>
  </Request>
</ref:httpRequest>
```
Resolved body returned by the compiler:
```json
{ "user":"Test User" }
```

**Request with only an authorization header** (no body):

```xml
<ref:httpRequest uri="http://somedomain.net/api/endpointWithAuth">
  <Request>
    <Headers>
      <Header name="Authorization">Basic S29yZWliYTo3ZjNiMGMxYzYwN2M0M2VhODg3ODY3ZjY3OGI2ZDVlMQ==</Header>
    </Headers>
  </Request>
</ref:httpRequest>
```

## Gotchas

- **The response replaces the reference.** Whatever the service returns becomes the value of `ref:httpRequest`. Design the endpoint to return exactly the string/JSON you want substituted in.
- **Always `encode="json"` values going into a JSON body.** Without it, a value containing a quote, backslash, or newline produces invalid JSON. The compiler's resolved-body output is the easiest way to confirm your escaping is correct.
- **Body-only mode is literal.** If you skip `<Request>`, every character inside the element — whitespace and newlines included — is part of the body. Be deliberate about formatting.
- **`Content-Type`/`Keep-Alive` headers are rejected.** Use the `type` attribute to set the content type, not a header.
- **Timeout is capped at 15 seconds.** Long-running work belongs in a delayed/asynchronous pattern, not a synchronous HTTP reference.

## Notes

### HMAC

When you *receive* an `httpRequest` from Aprimo, validate it with the HMAC header to confirm authenticity. Take the secret from the `.hmacSecret` setting, compute the base64-encoded HMAC-SHA256 of the HTTP body, and compare it to the header value. They should match for genuine messages.

Sample C# to compute the HMAC:

```csharp
using System.Security.Cryptography;
using System.Text;

static string CalculateAprimoWebhookHMAC(string secret, string httpBodyContent)
{
    var payload = Encoding.UTF8.GetBytes(httpBodyContent);
    var key = Encoding.UTF8.GetBytes(secret);
    using (var hash = new HMACSHA256(key))
    {
        return System.Convert.ToBase64String(hash.ComputeHash(payload));
    }
}
```

### Troubleshooting

For each HTTP request to your service, Aprimo includes a `referenceId` request header with a unique ID. When your service calls back into DAM while handling the request, include the **same** `referenceId` header on that DAM request. This lets Aprimo Product Support correlate the outbound HTTP request with the inbound DAM request when investigating issues.

**Example flow:** A user changes a field → the field's default-value reference fires an HTTP request to your webservice, carrying `referenceId` (e.g. `523af4bb-1968-4f60-ad4c-afda01694e75`) → your service logs that ID and calls the REST API to check the user, echoing the same `referenceId` → the API confirms the user → your service returns the user name to DAM. Both requests appear in the Aprimo log under the same ID.
