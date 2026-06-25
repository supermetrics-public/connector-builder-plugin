---
name: connector-setup
description: Use when the supermetrics CLI is not yet installed, when its API key is not yet configured, when no JSON Schema validator is on PATH, or when basic supermetrics commands fail with "command not found" or exit code 65.
---

# Connector Builder — one-time setup

Run this once per machine before using any other skill in the
`supermetrics-connector-builder` plugin. The plugin's other skills
assume the three pre-requisites below are already in place.

## Pre-requisites

1. **Supermetrics CLI** installed and on `PATH`.
2. **API key** configured (created in Supermetrics Hub, registered
   with the CLI).
3. **JSON Schema validator** (`check-jsonschema` or `ajv-cli`) on
   `PATH`.

## Steps

### 1. Install the CLI

Detect what's available:

```bash
command -v supermetrics && supermetrics --version
```

If missing, install with **Homebrew (recommended)**:

```bash
brew install supermetrics-public/tap/supermetrics
```

Other install paths (binary, `go install`, source) are in
`${CLAUDE_PLUGIN_ROOT}/docs/cli-reference.md §1`. Re-run the detect step until
`supermetrics --version` prints a version.

### 2. Create and register an API key

1. Open https://hub.supermetrics.com/api-key-management and have the
   user create a fresh key. Surface the URL clearly in the chat — do
   not assume the user knows where to go.
2. Run `supermetrics configure` and have the user paste the key when
   prompted. Alternative: `export SUPERMETRICS_API_KEY=…` in their
   shell profile.

Verify:

```bash
supermetrics profile show
```

Exit 0 with profile data printed = success. Exit 65 = key missing,
wrong, or expired; go back to step 1 with a new key.

> **Why API key, not OAuth?** OAuth would interrupt agent loops with
> a browser flow. API-key auth is non-interactive and stable across
> sessions — the right choice when the CLI is being driven by an
> agent.

### 3. Install a JSON Schema validator

Detect:

```bash
command -v check-jsonschema || command -v ajv
```

If neither is present, install **one** (prefer the first):

```bash
pipx install check-jsonschema     # recommended — works with no flags
npm install -g ajv-cli            # alternative — requires --strict=false
```

Verify:

```bash
check-jsonschema --version    # or: ajv --version
```

The `connector-configuring` skill uses whichever it finds.

## Verification (end-to-end)

```bash
supermetrics --version
supermetrics profile show
check-jsonschema --version || ajv --version
```

All three exit 0 → setup is complete. Hand control back to
`connector-builder` (or whichever skill the user is already in).

## Common setup failures

| Symptom | Fix |
|---|---|
| `supermetrics: command not found` after install | Restart shell, or run `brew doctor` to check PATH. |
| `supermetrics profile show` returns exit 65 | API key not registered, wrong, or revoked. Get a new one from Hub, re-run `supermetrics configure`. |
| `check-jsonschema: command not found` after `pipx install` | `pipx` install dir not on PATH. Run `pipx ensurepath`, restart shell. |
| Network errors during install | Corporate proxy / VPN. User to resolve manually; not in scope for this skill. |

## What this skill does NOT do

- It does **not** authenticate via OAuth even if the CLI defaults to
  it. The plugin standardizes on API-key auth.
- It does **not** install MCP servers or other auxiliary tools — the
  plugin only depends on the three pre-requisites above.
- It does **not** touch `~/.config/supermetrics/config.json` directly;
  always go through `supermetrics configure`.

## Deeper reference

- `${CLAUDE_PLUGIN_ROOT}/docs/cli-reference.md §§1–2` — install paths and authentication detail.
- `${CLAUDE_PLUGIN_ROOT}/docs/cli-reference.md §13` — why a JSON Schema validator matters.
