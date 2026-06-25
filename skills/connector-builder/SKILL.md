---
name: connector-builder
description: Use when the user mentions Supermetrics, Connector Builder, or any task involving authoring, editing, validating, running, or troubleshooting a Supermetrics Connector.
---

# Connector Builder for Supermetrics

Orientation for building a Supermetrics Connector via the `supermetrics` CLI.

## Vocabulary

- **Connector** — integration with one Data Source.
- **Configuration** — JSON file (`config.json`; YAML later) defining a Connector.
- **Report Type** — one retrievable dataset: request + parsing + Fields (Metrics + Dimensions).
- **Connection** — authorization state for one Connector ↔ Data Source.
- **Account** — scope under one Connection (e.g. one of many ad accounts).
- **Secret** — value referenced by ID in the config; lives in the CLI's secrets store, never inline.

## Workflow

| Phase | Owning skill |
|---|---|
| 1. Scope — what data, which API | `connector-scoping` |
| 2–3. Tech Overview + verify | `connector-technical-overview` |
| 4. Translate to `config.json` | `connector-configuring` + `connector-config-*` |
| 5. Run + compare to samples | `connector-validation` |
| Diagnose misbehavior | `connector-troubleshooting` |
| Upload logo | `connector-logo` |

## Project folder

One per Connector, created under the user's CWD: `<connector-name>/` holding `technical-overview.md`, `samples/`, `config.json`, `.env`, `.team-id`, `.gitignore`.

## CLI not ready?

Use `connector-setup` if any `supermetrics` command fails with "command not found" or exit 65.

## Deeper reference

- `${CLAUDE_PLUGIN_ROOT}/docs/connector-core-knowledge.md` — concepts.
- `${CLAUDE_PLUGIN_ROOT}/docs/cli-reference.md` — commands.
- `${CLAUDE_PLUGIN_ROOT}/docs/docs-portal.md` + `${CLAUDE_PLUGIN_ROOT}/docs/docs-links.md` — live doc lookup.
