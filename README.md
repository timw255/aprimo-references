# aprimo-references

A [Claude Skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) for writing
Aprimo reference expressions.

Aprimo references are an XML DSL (`<ref:...>` tags) that runs against a digital-asset-management
record and produces a string. They are used in field default values and rule actions.

## What it does

The skill gives Claude three things:

- **The tag documentation** — `docs/`, one file per tag (`compare.md`, `datetimeeval.md`,
  `switch.md`, …), plus `reference/` for cross-cutting topics: the catalog, the computational model,
  operators and logic, field configuration, automation patterns, and debugging.
- **The compiler** — a bundled binary in `bin/` that lints, executes, and analyzes a reference
  locally. Claude runs the reference through it instead of guessing, so what you get back has been
  checked rather than recalled.
- **A workflow** — `SKILL.md` tells Claude not to hand over a reference it hasn't run through the
  compiler.

## Install

Download `aprimo-references.zip` from [Releases](../../releases) and unzip it into your skills
directory. One universal bundle works everywhere - Windows, macOS, Linux, and claude.ai in the
browser (its sandbox is Linux x86-64): `bin/` carries all three compiler binaries and the skill
uses the right one for the host.

For Claude Code, that directory is `~/.claude/skills/` (personal) or `.claude/skills/` (project).

## Using the compiler directly

The reference XML goes in on stdin. Use `bin/transpiler.exe` on Windows, `bin/transpiler` on macOS,
`bin/transpiler-linux` on Linux.

```bash
# Is it valid?
echo '<ref:record fieldName="Title" out="value" store="@title"/><ref:text out="@title"/>' \
  | bin/transpiler -mode lint

# What does it output?
echo '<ref:text out="Hello"/>' | bin/transpiler -mode execute

# Run it against your own record data
echo '<ref:record fieldName="Status" out="value" store="@st"/><ref:text out="@st"/>' \
  | bin/transpiler -mode execute -context-json '{"records":[{"fields":{"Status":"Approved"}}]}'
```

Other modes: `debug` (variable state, optionally at `-break N`), `analyze` (structure report),
`dump-context` (the default record context). See `reference/compiler.md`.

## Contents

```
SKILL.md      workflow and rules Claude follows
docs/         one file per ref: tag
reference/    catalog, computational model, patterns, debugging, compiler usage
bin/          compiler binaries + default_context.json
```
