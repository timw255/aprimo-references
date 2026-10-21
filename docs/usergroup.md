# User Group reference (`ref:usergroup`)

Retrieves a property of a user group, looked up by id, name, or search expression.

## Syntax
```xml
<ref:usergroup id="id" out="out"/>
<ref:usergroup name="name" out="out"/>
<ref:usergroup search="expression" out="out"/>
```

## Attributes
| Attribute | Required | Description |
|---|---|---|
| `out` | yes | The group property to return. Must be one of the lowercase values below. |
| `id` | no | Look up the group by Id. |
| `name` | no | Look up the group by name. |
| `search` | no | Look up the group by a search expression. |

Use at most one of `id`, `name`, or `search`.

### Valid `out` values
The compiler enforces a closed, **lowercase** set. Any other value fails `lint`:

`id`, `name`, `tag`, `createdonlocal`, `createdonutc`, `createdby`, `modifiedonlocal`, `modifiedonutc`, `modifiedby`

| Value | Type | Description |
|---|---|---|
| `id` | Guid | Id of the user group. |
| `name` | String | Name of the user group. |
| `tag` | String | The Tag property. |
| `createdonlocal` / `createdonutc` | DateTime | Creation date/time (local / UTC). |
| `createdby` | Guid | Id of the user that created the group. |
| `modifiedonlocal` / `modifiedonutc` | DateTime | Last-modified date/time (local / UTC). |
| `modifiedby` | Guid | Id of the user that last modified the group. |

## Examples

Get a group's id by name:
```xml
<ref:usergroup name="Everyone" out="id"/>
```
Produces: `"00db70f9-2fd2-417c-b898-337b7a26641a"`

Read the Tag of a group:
```xml
<ref:usergroup name="Designers" out="tag"/>
```
Produces: `"creative_team"`

Look up a group by id and read its name:
```xml
<ref:usergroup id="marketing-guid-789" out="name"/>
```
Produces: `"Marketing"`

Read who created a group:
```xml
<ref:usergroup name="Designers" out="createdby"/>
```
Produces: `"admin-user"`

Capture silently with `store`, then emit:
```xml
<ref:usergroup name="Everyone" out="id" store="@grp"/><ref:text out="@grp"/>
```
Produces: `"00db70f9-2fd2-417c-b898-337b7a26641a"`

## You cannot list a group's members

There is no reference that enumerates the users of a group. `ref:user search="group.name =
'Marketing'"` is a **valid** search (the `group.` prefix is a membership subquery), but like every
`ref:user` lookup it must resolve to **exactly one** user - with two or more members it throws
`More than one matching item has been found.`. So it can pick *the* member of a
one-person group, never produce a list.

What works instead:

- **A distribution list in a setting** - a custom Text setting holding the addresses
  (`SecurityTeam` = `a@x.com;b@x.com`), read with `<ref:setting name="SecurityTeam"/>` in the
  e-mail `<recipients>` slot. Editable by an admin without touching the rule.
- **A User List field on the record** - iterate it:
  `<ref:record fieldName="Reviewers" out="value" store="@ids"/><ref:foreach in="@ids" storeitem="@id" join=";"><ref:user id="@id" out="email"/></ref:foreach>`.
- **Subscribers** - the Send e-mail action's *Subscribers list* target notifies the users following
  the record, via `<ref:subscribers out="email"/>` (only valid in that action).

Searchable `ref:usergroup` properties are `name`, `id` and `tag` (plus the audit dates/users). A
property the group does not have - including a dotted subquery such as `user.name` - throws
`The specified field 'user' does not exist.`; a valid property that matches nothing throws
`No user group has been found matching the specified criterion.`

## Gotchas
- **`out` values are lowercase and case-sensitive at runtime.** `out="Id"` passes `lint` but resolves to an **empty string** when executed; use `out="id"`. Trigger the lint error with an invalid value (e.g. `out="ZZZ"`) to print the full valid set.
- **A lookup that matches no group THROWS** - `No user group has been found matching the specified
  criterion.` It does not resolve to an empty string. An uncaught throw aborts the whole reference,
  so guard any lookup that might miss:
  `<ref:object store="@grp"><ref:catch ex="System.Exception" out="unassigned"><ref:usergroup name="@nm" out="name"/></ref:catch></ref:object>`
- A `store` attribute captures without emitting; pair with an ungated `ref:text` to print. See `../reference/patterns.md`.

## See also
- `user.md` — `out="groupnames"`/`out="groupids"` returns the groups a user belongs to.
- `../reference/patterns.md` — store/emit patterns.
