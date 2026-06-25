# Supermetrics Connector Builder for Claude Code

**For Supermetrics customers and partners building Connectors with
Claude Code.** This plugin loads a set of skills that guide Claude
through a proven 5-phase workflow — define what data you want,
research the source's API, produce a working Configuration, and
validate it against real data — all by driving the official
`supermetrics` CLI. You stay in your terminal; you don't have to
switch to the Connector Builder web UI for routine authoring.

---

## What you get

A set of Claude skills that load on demand when you're working on a
Supermetrics Connector. They cover:

- Defining what data you want and from which API.
- Researching the Data Source's documentation and confirming with
  real requests that the data is actually there.
- Writing the Connector Configuration the `supermetrics` CLI expects
  (with pre-flight checks and schema validation before every save).
- Running the saved Connector and comparing its output to your
  reference samples.
- Reading CLI logs to diagnose problems.
- Managing secrets, uploading a Connector logo, and other utility
  tasks.

You stay in Claude Code the whole time. No need to switch to the
Connector Builder web UI for routine authoring.

## Who this is for

- Supermetrics **customers** building Connectors — both low-code
  authors and engineers comfortable in a terminal.
- **Partners and consultants** building Connectors for clients across
  multiple Supermetrics teams.

> Note: this plugin is for customers and partners. Supermetrics
> internal connector-catalog maintainers use a separate internal
> tool.

---

## Pre-requisites

These need to be in place **before** invoking any skill in this
plugin. The skills assume them as a baseline.

### 1. Install the Supermetrics CLI

```bash
brew install supermetrics-public/tap/supermetrics
```

Other install paths (binary, `go install`, source) are in the
[CLI repo](https://github.com/supermetrics-public/supermetrics-cli).

### 2. Create an API key and configure the CLI

1. Go to [Supermetrics Hub → API Key Management](https://hub.supermetrics.com/api-key-management)
   and create a key.
2. Run `supermetrics configure` and paste the key
   (or `export SUPERMETRICS_API_KEY=…` in your shell profile).
3. Verify with `supermetrics profile show`.

We recommend API-key auth (rather than the CLI's default OAuth flow)
because the skills work best when the CLI is non-interactively
authenticated.

### 3. Install a JSON Schema validator

Either of:

```bash
pipx install check-jsonschema     # recommended — works out of the box
npm install -g ajv-cli            # alternative — needs --strict=false
```

The configuring skill uses whichever it finds on your PATH.

---

## Install the plugin

> **Coming soon.** This plugin is in pre-release. Install instructions
> will be added here once distribution is finalized. If you've found
> this repo and want to try the plugin against your own Connectors
> early, the maintainers can help — open an Issue and we'll point you
> at the current install path.

---

## Quickstart

Start Claude Code in any directory you want your Connector projects
to live under (for example `~/connectors/`). Then drop into one of
the three entry paths below — the plugin's skills auto-load when
your prompt matches.

> The exact prose below shows the shape of an interaction. Actual
> dialogue will vary; the plugin's skills negotiate the details with
> you turn by turn.

### Path A — start from a working script (the headline case)

If you already have a script (vibe-coded with an AI tool, written by
hand, or anything in between) that pulls the data you want from the
source, this is the fastest path. Drop it next to your Claude Code
session and say:

> _"Here's a Python script that already pulls campaign data from the
> Acme Ads API. Turn it into a real Supermetrics Connector."_

The agent reads the script, extracts the endpoints, auth method, and
fields, and proposes a scope statement. From there it walks you
through verifying with real requests (Phase 3), writing the
Connector Configuration (Phase 4), and validating with `supermetrics
queries execute` (Phase 5).

Hard-coded secrets in the script get moved into the secrets store —
not copied anywhere else, not echoed back at you.

### Path B — start from scratch

If you don't have an artifact yet, describe what you want:

> _"I want to build a Supermetrics Connector for the Acme Ads API
> that fetches typical campaign efficiency data."_

The agent will ask the minimum it needs to fix the scope (which API
family, which data, what granularity), then drive Phases 2–5 the
same way as Path A.

### Path C — edit an existing Connector

If you already have a saved Connector you want to fix or extend:

> _"Edit my Acme Connector — pagination is broken on the Campaign
> Performance report."_

The agent runs `supermetrics connector-builder list`, finds your
Connector, downloads its current Configuration, and seeds a local
project folder from it. Then it jumps straight to the part of the
workflow that needs work (Phase 4 for fixes, Phase 5 for
re-validation, troubleshooting for diagnosis).

### What gets created

In all three paths, the plugin creates a `<connector-name>/` folder
under your current working directory and uses it as the working
context. See [Per-Connector project folder](#per-connector-project-folder)
below for the file layout.

---

## What each skill does

| Skill | When it triggers |
|---|---|
| `connector-builder` | Always relevant when working on a Supermetrics Connector — vocabulary + workflow map. |
| `connector-setup` | First-time setup of the CLI / API key / validator, or when basic CLI commands fail. |
| `connector-scoping` | Starting a new Connector, productizing an existing script, or editing an existing Connector. |
| `connector-technical-overview` | Researching the Data Source's API and producing a verified Technical Overview document. |
| `connector-configuring` | Writing or modifying the Connector Configuration file the CLI accepts. |
| `connector-validation` | Running a saved Connector and confirming it returns the expected data. |
| `connector-troubleshooting` | A Connector behaves wrong: wrong data, runtime error, auth failure, empty results. |
| `connector-logo` | Uploading or replacing the Connector's logo image. |
| `connector-config-auth` | Configuring authentication and Connection (auto-invoked by `connector-configuring`). |
| `connector-config-accounts` | Configuring how Accounts are listed and selected (auto-invoked). |
| `connector-config-report-type` | Adding or modifying one Report Type (auto-invoked, once per Report Type). |
| `connector-config-response-handling` | Configuring default response parsing / pagination / error handling (auto-invoked). |

The full vocabulary, the 5-phase workflow in detail, and the CLI
command surface are documented in the plugin's
[`docs/`](./docs) folder. The plugin loads them on demand via the
skills.

---

## The 5-phase build workflow

The skills are organized around a workflow that's evolved from real
experience building Connectors:

1. **Scope** — what data, from which API of the Data Source.
2. **Technical Overview** — what endpoints, what auth, what fields,
   what errors, what rate limits. Captured in a document.
3. **Verify** — actually hit the endpoints with real requests and
   confirm the data matches what you expect.
4. **Configure** — translate the Technical Overview into the
   Configuration file the CLI accepts.
5. **Validate** — run the saved Connector via `supermetrics queries
   execute` and compare its output to the samples from Phase 3.

The skills nudge you to follow this order. Skipping ahead (especially
to Phase 4 without a Technical Overview) is the most common way to
end up rewriting your Configuration.

---

## Per-Connector project folder

Each Connector you build gets its own working folder under wherever
you started your Claude Code session. The skills create and populate it.

```
acme-reporting/
  technical-overview.md     # Phase 2 artifact
  samples/                  # Phase 3 raw responses
  config.json               # Phase 4 Configuration (config.yaml once supported)
  logs/                     # optional Phase 5 artifacts
  .env                      # local secret values — NEVER commit
  .team-id                  # cached --team-id for this project
  .gitignore                # at minimum: .env, .team-id
```

**Never commit `.env`** to version control — it holds your real secret
values. The skills add it to `.gitignore` automatically.

---

## Troubleshooting basics

A few common setup / install issues. For diagnosing a Connector
that behaves wrong — wrong data, runtime errors, auth failures —
ask Claude to troubleshoot it; the `connector-troubleshooting`
skill loads automatically and walks through CLI logs, exit codes,
and the secrets-vs-auth distinction.

Common setup situations:

- **`supermetrics: command not found`** — re-run the install in
  Pre-requisites §1.
- **CLI returns exit code 65** — your API key isn't configured or has
  expired. Run `supermetrics configure` again with a fresh key from
  [Supermetrics Hub](https://hub.supermetrics.com/api-key-management).
- **A Connector you just saved errors with auth failure** — first
  check that all secrets referenced in your Configuration are
  registered via `supermetrics connector-builder-secrets list`. A
  missing secret reads as a generic auth failure.
- **Pre-flight check or schema validation fails** — the agent will
  show you the offending field. The `$schema` reference at the top
  of your `config.json` also gives editors (VS Code, JetBrains)
  inline autocomplete and validation.

For deeper diagnosis, ask Claude to "troubleshoot my <name>
Connector" — the `connector-troubleshooting` skill walks through
CLI logs, exit codes, and the secrets-vs-auth distinction.

---

## Authoritative sources the agent uses

- **Connector Configuration JSON Schema** —
  https://api.supermetrics.com/connector_builder/v1/schema/v1
  (cached locally for the session).
- **Supermetrics docs portal** (primary source for how-tos and
  reference) — https://docs.supermetrics.com/docs/connector-builder.
- **Internal Confluence space `CC`** (secondary source for technical
  depth) — https://smdocs.atlassian.net/wiki/spaces/CC/overview.
- **Supermetrics CLI** — https://github.com/supermetrics-public/supermetrics-cli.

The agent fetches from these on demand. We don't vendor any docs in
the plugin so you always see current content.

---

## Versioning

Plugin versions follow a **calendar scheme**: `YYYY.MM.N` (e.g.
`2026.06.0`). Releases are tagged on GitHub.

The plugin re-downloads the JSON Schema (cached daily) and references
the docs portal live, so most changes to the platform are picked up
without a new plugin release.

---

## Support and feedback

Open an issue at
[github.com/supermetrics-public/connector-builder-plugin/issues](https://github.com/supermetrics-public/connector-builder-plugin/issues)
for bug reports, feature requests, or doc gaps.

When reporting an issue, including the following helps a lot:

- The exact prompt that triggered the unexpected behavior.
- Which skill (if any) the agent loaded.
- The exit code and stderr from any failing `supermetrics` CLI
  command (see your `<project>/logs/` folder).
- The plugin version (`supermetrics-connector-builder` line in
  `/plugin list` output).

For Supermetrics product / account / billing questions, use the
existing Supermetrics support channel rather than this repo.

---

## License

Apache License 2.0 — see [LICENSE](LICENSE) for the full text.

---

## Contributing

This plugin is currently **maintained internally by Supermetrics**.
External pull requests will be closed with a pointer back here —
contributions are welcome via the **Issues** tracker
(bug reports, feature requests, documentation fixes).

The contribution model may change as the plugin matures; check this
section for updates.
