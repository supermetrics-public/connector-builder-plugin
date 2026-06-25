---
name: connector-configuring
description: Use when a Technical Overview for a Supermetrics Connector exists and the Connector Configuration file is missing, incomplete, or needs changes.
---

# Phase 4 — Configure

You are the **translator**: turn the verified Technical Overview into
the Connector Configuration that `supermetrics connector-builder
update` accepts. Most decisions are already made in the Overview; if
they aren't, go back to Phase 2 rather than guessing here.

## Pre-conditions

Before starting, confirm:

- `<project>/technical-overview.md` exists, every required section
  has a status emoji, and Summary verdict is _Feasible_ or _Feasible
  with caveats_.
- `<project>/samples/` has at least one verified sample per Report
  Type.
- `<project>/.team-id` exists (if not, ask the user or run
  `supermetrics connector-builder list` to detect).
- A JSON Schema validator is on `PATH` (`check-jsonschema` or `ajv`).
  If not, hand off to `connector-setup`.

## File format

Today the Configuration file is `<project>/config.json`. The platform
is moving to YAML; once the CLI accepts `config.yaml`, detect format
from the file extension and read/write accordingly. Pre-flight checks
operate on parsed content, so they work for both.

Always include a `$schema` reference at the top of the file:

```json
{
  "$schema": "https://api.supermetrics.com/connector_builder/v1/schema/v1",
  …
}
```

This gives editors (VS Code, JetBrains, etc.) autocomplete and inline
validation for free.

## The build loop

```
write/edit → pre-flight checklist (Layer A)
           → schema validation (Layer B)
           → connector-builder update
           → read errors → fix → repeat
```

Every save is effectively a publish — there is no draft mode. Trust
Layers A and B to keep saves clean.

### Layer A — Pre-flight checklist (agent-applied)

Before every `update`, walk this list and fix anything that fails:

1. **Parses.** The file is valid JSON (or YAML).
2. **Required sections present.** At minimum the Configuration has
   the authentication / Connection root, an Accounts section (or an
   explicit "connection-as-account" note), Reporting with at least
   one Report Type, and a Secrets declaration if any secrets are
   referenced.
3. **Secrets declared = secrets used.** Every secret ID referenced
   anywhere in the Configuration appears in the root `secrets`
   object, and vice versa (no unused declarations).
4. **Secrets present in the store.** Run `supermetrics
   connector-builder-secrets list --team-id "$(cat .team-id)"
   --connector-identifier <ds-id>` and confirm every declared ID is
   registered. A missing secret will surface as a generic auth
   failure at runtime (see `connector-troubleshooting`).
5. **Field IDs resolve.** Every field referenced by a Report Type is
   declared in that Report Type's field list.
6. **Cross-references resolve.** Account selectors, response paths,
   etc., point at things that exist in the Configuration.

These are checks you apply, not a script you run.

### Layer B — Schema validation (third-party tool)

```bash
# preferred — works out of the box:
check-jsonschema --schemafile "$SCHEMA_CACHE" <project>/config.json

# alternative — needs --strict=false:
ajv validate -s "$SCHEMA_CACHE" -d <project>/config.json --spec=draft2019 --strict=false
```

`$SCHEMA_CACHE` is the locally-cached schema; fetch and cache per the
recipe in `${CLAUDE_PLUGIN_ROOT}/docs/cli-reference.md §14` (~/.cache/supermetrics-connector-builder/
connector-schema.v1.json, 24-hour TTL).

Schema validation is necessary but **not sufficient** — the schema
is permissive at the root (empty `{}` passes). Layer A catches the
structural mistakes the schema misses.

### Save via the CLI

Only after both layers pass:

```bash
supermetrics connector-builder update \
  --team-id "$(cat .team-id)" \
  --connector-identifier <ds-id> \
  --connector-file '{"connector": {"name": "<Name>", "description": "<Description>"}}' \
  --configuration-file <project>/config.json
```

`--connector-file` is passed inline as a JSON string; only
`--configuration-file` lives on disk.

## When to fan out to a Tier-3 sub-skill

The Configuration breaks into functional tasks. For each, call the
corresponding sub-skill rather than authoring inline:

| Task | Sub-skill |
|---|---|
| Authentication + Connection + wiring secrets | `connector-config-auth` |
| Listing and selecting Accounts | `connector-config-accounts` |
| One Report Type (request, response parsing, fields) | `connector-config-report-type` — invoke once per Report Type |
| Default response handling (pagination, error handling, shared parsing) | `connector-config-response-handling` |

A typical sequence for a new Connector: auth → accounts → one report
type → default response handling → additional report types →
pre-flight → schema validate → save.

## First save (new Connector)

If the Connector doesn't exist yet, `create` it first to get an
identifier:

```bash
supermetrics connector-builder create \
  --team-id "$(cat .team-id)" \
  --connector-file '{"connector": {"name": "<Name>", "description": "<Description>"}}' \
  --configuration-file <project>/config.json
```

The platform seeds new Connectors with a valid boilerplate, so the
first `create` will not error on missing structure. Capture the
returned `ds-id` and write it to `<project>/.ds-id` for later
`update` and `queries execute` calls.

## After every successful save

- Save the saved Configuration (e.g. `supermetrics connector-builder
  get …`) and diff against the local file — if it differs, the
  server normalized something. Note any normalization in the user's
  next interaction.
- Hand off to `connector-validation` when the Configuration is
  considered ready to test against real data.

## Common mistakes

| Mistake | Fix |
|---|---|
| Skipping Layer A because Layer B passed | The schema is permissive. Always walk the checklist. |
| Inventing structure not present in the Tech Overview | If a required detail isn't in the Overview, go back to Phase 2 — don't guess. |
| Inlining a secret value into the config "just for testing" | Never. Always register via `connector-builder-secrets create`. |
| Editing both auth and reporting in one round before saving | Save after each functional change so error messages stay focused. |
| Forgetting that every save is a publish | Treat each `update` accordingly. No throwaway connectors — trust the layers. |

## Deeper reference

- `${CLAUDE_PLUGIN_ROOT}/docs/connector-core-knowledge.md §7 Phase 4` — workflow context.
- `${CLAUDE_PLUGIN_ROOT}/docs/cli-reference.md §§4, 13, 14` — `connector-builder` commands, validation strategy, schema caching.
- `${CLAUDE_PLUGIN_ROOT}/docs/docs-links.md` — section: "Core configuration concepts."
