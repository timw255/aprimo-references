# User reference (`ref:user`)

Retrieves a property of the current user, or of a user looked up by id, name, or search expression.

## Syntax
```xml
<ref:user out="out"/>
<ref:user id="id" out="out"/>
<ref:user name="name" out="out"/>
<ref:user search="expression" out="out"/>
```

With no `id`, `name`, or `search`, the reference resolves against the **current** user (the user the rule is running for).

## Attributes
| Attribute | Required | Description |
|---|---|---|
| `out` | yes | The user property to return. Must be one of the lowercase values below. |
| `id` | no | Look up the user by Id (a GUID). |
| `name` | no | Look up the user by name/Login ID. Matches the display name or the username. |
| `search` | no | Look up the user by a search expression. |

Use at most one of `id`, `name`, or `search`. Omit all three for the current user.

### Valid `out` values
The compiler enforces a closed, **lowercase** set. Any other value fails `lint`:

`authtoken`, `createdonlocal`, `createdonutc`, `createdby`, `department`, `email`, `groupids`, `groupnames`, `id`, `languageid`, `languageidforui`, `lastsuccessfullogondate`, `modifiedonlocal`, `modifiedonutc`, `modifiedby`, `name`, `roles`, `tag`

| Value | Type | Description |
|---|---|---|
| `id` | Guid | Id of the user. |
| `name` | String | Name / Login ID of the user. |
| `email` | String | E-mail address of the user. |
| `groupnames` | List\<String\> | User groups the user belongs to (comma-joined when emitted as text). |
| `groupids` | List\<Guid\> | Ids of the user groups the user belongs to. |
| `department` | String | The user's department. |
| `roles` | List\<String\> | The user's roles. |
| `tag` | String | The Tag property. |
| `authtoken` | String | Authorization token usable in HTTP request references. |
| `languageid` | Guid | Default language used when accessing fields. |
| `languageidforui` | Guid | Selected UI language. |
| `lastsuccessfullogondate` | DateTime | Last UTC date the user logged in successfully (null if never). |
| `createdonlocal` / `createdonutc` | DateTime | Creation date/time (local / UTC). |
| `createdby` | Guid | Id of the user that created this user. |
| `modifiedonlocal` / `modifiedonutc` | DateTime | Last-modified date/time (local / UTC). |
| `modifiedby` | Guid | Id of the user that last modified this user. |

## Examples

Current user's id:
```xml
<ref:user out="id"/>
```
Produces: `"00000000-0000-0000-0000-000000000001"`

Current user's email:
```xml
<ref:user out="email"/>
```
Produces: `"test@example.com"`

Current user's group memberships (a list is joined with commas when emitted as text):
```xml
<ref:user out="groupnames"/>
```
Produces: `"Designers,Marketing"`

Look up a specific user by name and read a property:
```xml
<ref:user name="Administrator" out="email"/>
```
Produces: `"admin@example.com"`

Look up a user by id:
```xml
<ref:user id="00000000-0000-0000-0000-000000000002" out="name"/>
```
Produces: `"Administrator"`

Capture a value silently with `store`, then emit it explicitly:
```xml
<ref:user out="name" store="@me"/><ref:text out="@me"/>
```
Produces: `"Test User"`

## Gotchas
- **`out` values are lowercase and case-sensitive at runtime.** `out="Id"` and `out="Email"` *pass* `lint` (validation is case-insensitive) but resolve to an **empty string** when executed. Always write the value exactly as listed (`out="id"`, `out="email"`). Use `out="ZZZ"` to see the full valid set in the lint error.
- A `store` attribute captures the value **without emitting** it — nothing prints unless an ungated `ref:text` (or another emitting reference) outputs it. See `../reference/patterns.md`.
- `name` matches both the display name (`"Administrator"`, `"Test User"`) and the username (`"testuser"`); an arbitrary handle that is neither resolves to empty.
- `groupnames`/`groupids`/`roles` are lists. Emitted directly as text they are comma-joined; pair with `store` + `ref:foreach`/`ref:count` to operate on the items.
- A lookup that matches no user resolves to an empty string rather than erroring.

## See also
- `../reference/patterns.md` — store/emit, lists, and foreach patterns.
- `usergroup.md` — look up the groups themselves.
