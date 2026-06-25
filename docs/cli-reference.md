# Supermetrics CLI Reference (for Connector skills)

Companion to `connector-core-knowledge.md`. Captures the **command surface**
of the Supermetrics CLI that Connector-related skills drive. Core knowledge
covers concepts and vocabulary; this doc covers commands and flags.

- Repo: https://github.com/supermetrics-public/supermetrics-cli
- Canonical README: https://github.com/supermetrics-public/supermetrics-cli/blob/main/README.md

**Sync convention:** Anything marked _“NOT IN README”_ was not documented at
the time this file was distilled. Skills should fall back to
`supermetrics <command> --help` as the authoritative current signature, and
this file is updated when the README catches up.

---

## 1. Install

- **Homebrew** — `brew install supermetrics-public/tap/supermetrics`
  (works today, despite the README still saying "not yet available").
- **Binary** — GitHub Releases.
- **Go** — `go install github.com/supermetrics-public/supermetrics-cli/cmd/supermetrics@latest`
- **Source** — `git clone` + `make build`.
- **APT / YUM** — not yet available (per README).

## 2. Authenticate the CLI itself

⚠️ This is the CLI's auth to the Supermetrics backend — **distinct from a
Connector's own auth to a Data Source**. Skills should keep the two
mental models separated when talking to users.

Two methods:

- **OAuth (CLI default recommendation):** `supermetrics login` opens a
  browser (Google / Microsoft).
- **API key:** `supermetrics configure` (interactive) or set
  `SUPERMETRICS_API_KEY=…`. Keys are created and rotated in
  **Supermetrics Hub**: https://hub.supermetrics.com/api-key-management.

### Pre-requisite for using these skills

Although the CLI itself recommends OAuth, the skills in this plugin are
designed to assume **API-key auth is already configured** before the
agent runs. This avoids browser-flow interruptions mid-workflow and
gives the agent a single, stable way to confirm the CLI is ready.

Recommended one-time setup the user does **before** invoking any skill
in this plugin:

1. Create an API key at https://hub.supermetrics.com/api-key-management.
2. Run `supermetrics configure` and paste the key (or
   `export SUPERMETRICS_API_KEY=…` in their shell profile).
3. Verify with `supermetrics profile show` (or any read-only command).

Each skill's SKILL.md lists this as a pre-requisite at the top. The
plugin's customer-facing README repeats the same three steps in its
"Getting started" section.

Credential resolution order (highest first):

1. `--api-key` flag
2. `SUPERMETRICS_API_KEY` env var
3. OAuth token from CLI config (auto-refreshed)
4. API key from CLI config

CLI config file: `~/.config/supermetrics/config.json` (mode `0600`).

## 3. Profiles (multi-environment for CLI auth)

```
supermetrics configure --profile work
supermetrics profile list
supermetrics profile use work
supermetrics profile show work
supermetrics profile delete staging
```

- Override per-command with `--profile staging`.
- Env var: `SUPERMETRICS_PROFILE`.

**Scope of profiles:** profiles isolate **CLI credentials only** (API key +
OAuth tokens). They do **not** scope Connector secrets — Connector secrets
are partitioned by `--team-id` on each command. A user who needs separate
dev/staging/prod Connector secrets uses different team IDs, not different
CLI profiles.

## 4. Connector-builder commands (Phases 4 and beyond)

| Subcommand                       | Purpose                              |
|---------------------------------|--------------------------------------|
| `connector-builder list`        | List connectors visible to the team  |
| `connector-builder get`         | Fetch a connector's metadata/config  |
| `connector-builder create`      | Create a new connector               |
| `connector-builder update`      | Update an existing connector         |
| `connector-builder delete`      | Delete a connector                   |

Repeated flags:

- `--team-id` — team scope.
- `--connector-identifier` — the connector's identifier (the `ds-id`,
  see §8). When the Connector is created via Connector Builder, this
  is auto-generated as a 5-character string; it can technically be
  shorter, but 5 chars is the standard.
- `--connector-file` — JSON with the connector **metadata wrapper**.
  Trivial — just `name` and `description`:

  ```json
  { "connector": { "name": "My Connector", "description": "What it does" } }
  ```

  The same JSON can be passed as a **plain string** instead of via a
  file; the string form is preferred. Skills should generate this
  inline rather than maintaining a separate `connector.json` file in
  the per-connector project folder.
- `--configuration-file` — JSON file with the actual Connector
  Configuration body (the multi-section thing described in core
  knowledge §4 / Phase 4). This **is** kept in the project folder
  (as `config.json` today, `config.yaml` later).

### No draft workflow

There is no published-vs-draft separation today. Every `update` (or
`create`) must carry a **valid** Configuration, or the CLI returns an
error and nothing changes.

When `create` succeeds, the new Connector starts with a **boilerplate
dummy Configuration** that is technically valid. The author then
iterates with `update`s until the real Configuration is in place.

**Implication for the configuring skill:** the build → save → fix loop
treats every save as a publish. The skill leans on the Layer-A
pre-flight checklist and the Layer-B schema validator (see §13) to
keep every save clean rather than introducing a throwaway-Connector
convention.

**NOT IN README:** explicit `publish`, `validate`, `lint`, `dry-run`,
version control, or rollback commands for connectors. (Validation today
= attempt to save and read the error — see §13 below for advice on
local pre-checks.)

## 5. Secrets — `connector-builder-secrets`

For Phase 4 / core knowledge §6.

| Subcommand                                | Purpose                                |
|-------------------------------------------|----------------------------------------|
| `connector-builder-secrets list`          | List registered secrets for a connector |
| `connector-builder-secrets create`        | Register a new secret                  |
| `connector-builder-secrets update`        | Replace value of an existing secret    |
| `connector-builder-secrets delete`        | Remove a secret                        |

Flags:

- `--team-id`, `--connector-identifier`, `--secret-name`
- `--secret-value` — if omitted, the CLI prompts with **masked terminal
  input** (preferred for interactive use; keeps the value out of shell
  history).

### Important: `.env` is _our_ convention, not a CLI feature

The CLI repository's own `.env` file is a build-time concern for CLI
developers (values injected via `-ldflags`). The CLI does **not** provide
a "load secrets from `.env`" command.

The "store secrets in a local `.env` during development" guidance in core
knowledge §6 is therefore a **skill-imposed convention**: it makes secret
values easy to keep handy while a user iterates on a connector. The
actual registration into the secrets store still happens one secret at a
time via `connector-builder-secrets create` (interactively prompted), or
through a small helper script the user wires up themselves.

Skills should not promise `.env` auto-loading.

## 6. Logs — `connector-builder-logs`

Primary diagnostic surface for Phase 5 and the troubleshooting skill.

| Subcommand                       | Purpose                              |
|----------------------------------|--------------------------------------|
| `connector-builder-logs list`    | List execution logs for a connector  |
| `connector-builder-logs get`     | Fetch a specific log entry           |

Detailed flag list and response shape are **not in the README**; defer
to `--help`. The troubleshooting skill should call `--help` once and
quote the relevant pieces inline when it's authored.

## 7. Logo — `connector-builder-logo`

| Subcommand                       | Purpose                        |
|----------------------------------|--------------------------------|
| `connector-builder-logo get`     | Fetch the current logo         |
| `connector-builder-logo upload`  | Upload (`--file`, PNG/JPG, max 5MB) |

## 7a. Datasource introspection — `datasource get`

**Not in the README**, but critical for Phase 5 validation. Returns
the runtime view of a saved Connector — including the **actual field
IDs that `queries execute` accepts**, which may differ from the IDs
declared in the Configuration JSON.

```bash
supermetrics datasource get \
  --data-source-id <ds-id> \
  --team-id <team-id>
  # optional: --sm-app-id <app-id>
```

When `queries execute --fields …` rejects a Configuration field ID
as unknown, the cause is almost always that the runtime ID is
different. Resolve by parsing the output of `datasource get` to find
the matching field, and pass that ID to `queries execute` instead.

## 7aa. Data-source Logins — `login-links` and `logins`

⚠️ **Terminology overlap:** the CLI's `login-links` and `logins`
commands manage authentication **from a Data Source to the Connector**
(what core knowledge §5.1 calls a **Connection**). This is **not**
the same as `supermetrics login` (§2), which is the user's CLI auth
to Supermetrics. Keep the two mental models separated when talking
to users.

### Why it matters

`queries execute` (§8) needs more than a saved Connector — it needs
an authenticated **Login** for that Connector's Data Source. Without
one, even a perfectly valid Configuration returns auth-failure.
Skills that drive validation must check for / create a Login before
attempting to run queries.

### `login-links create` — start the auth flow

```bash
supermetrics login-links create \
  --ds-id <ds-id> \
  --description "<internal label>" \
  --expiry-time <date-or-relative>
  # optional: --redirect-url <url>
  # optional: --require-username <existing-user>  (for renewing creds)
  # optional: --dry-run
```

Returns a **URL the user must open in a browser** to complete the
authentication flow (OAuth or whatever the Data Source uses).

The skill should:

1. Show the URL clearly to the user.
2. Tell them to open it and finish the authentication.
3. Wait for the user to confirm they're done.
4. Confirm via `logins list` that a new Login appeared.

### `login-links list` / `get` / `close`

| Subcommand              | Purpose                                       |
|-------------------------|-----------------------------------------------|
| `login-links list`      | List pending and used login links.            |
| `login-links get`       | Fetch one link's details.                     |
| `login-links close`     | Deactivate a link (stop accepting new logins).|

Use `close` after a successful login to prevent the URL from being
reused.

### `logins list` / `get`

| Subcommand    | Purpose                                                                |
|---------------|------------------------------------------------------------------------|
| `logins list` | Enumerate all authenticated Logins visible to the user.                |
| `logins get`  | Fetch a specific Login by ID.                                          |

The Login identifiers returned here are what `queries execute`
accepts via `--ds-users` / `--ds-user`.

## 7b. Account discovery — `accounts list`

**Not in the README**, but useful for validation when the user
hasn't memorized account IDs.

```bash
supermetrics accounts list \
  --ds-id <ds-id> \
  --ds-users <login-usernames>
  # optional: --cache-minutes <N>
```

Returns the Accounts a given login can see under the Connector. The
returned account IDs are what `queries execute --ds-accounts` expects.

## 8. Executing a Connector — `queries execute`

The Phase 5 workhorse. The CLI _runs_ a Connector and returns rows.

Example from the README:

```bash
supermetrics queries execute \
  --ds-id GAWA \
  --fields account_name,sessions \
  --start-date 2024-01-01 \
  --end-date 2024-01-31 \
  --ds-accounts list.all_accounts \
  [--all | --limit N] \
  [--max-rows N]
```

- `--ds-id` — the **Connector identifier** (same thing as the
  `--connector-identifier` flag in `connector-builder *`). Usually a
  5-character string assigned by Connector Builder at create time.
- `--fields` — comma-separated subset of fields exposed by the Report
  Type. Use the **runtime** IDs (see "Runtime field IDs" below), not
  the IDs declared in `config.json`.
- `--start-date` / `--end-date` — windowed reports.
- `--ds-accounts` — selector expression for which Accounts to fetch.
- `--all` / `--limit N` / `--max-rows N` — paging knobs (see below).
- Output: JSON (default), `table`, or `csv` (auto-flattens nested data).
- Long-running queries auto-poll with adaptive intervals (1s → 2s → 5s).

This is the command the build → execute → diagnose → fix loop hangs off
of.

### Paging: prefer `--max-rows`, not `--all`, for Connector Builder sources

For Connector Builder–authored sources, `--all` does **not** drive the
Connector's own pagination loop — it controls the platform-level "fetch
every available page" flag in a different context, and against a CB
source it can stop after the first page even when the underlying API
has more data.

Use `--max-rows N` to force pagination through the configured paging
mechanism: the platform iterates the Connector's pagination configuration
until either `N` rows are returned or the source API exhausts. This is
the only reliable way to verify pagination is correctly configured during
Phase 5 validation.

```bash
# Reliably exercise pagination across at least N rows:
supermetrics queries execute --ds-id <ds-id> --ds-user <login> \
  --fields … --start-date … --end-date … --ds-accounts <account> \
  --max-rows 1000
```

### Runtime field IDs

The `--fields` flag expects **runtime field IDs**, which are not the
same as the field IDs declared in `config.json`. Under schema v1:

```
runtime_field_id = <report_type_id> + "_" + <config_field_id>
```

Example: a Configuration field `clicks` inside Report Type `ad_perf`
is queried as `ad_perf_clicks`. Underscore separator, **not** dot.
The rule is deterministic — construct the runtime IDs from the
Configuration; you do not need to look them up.

Neither `datasource get` nor `connector-builder-logs *` displays the
prefix rule explicitly. The prefixed IDs do appear in the field
listing returned by `datasource get`, but they are not labeled as
"runtime-prefixed" — they're just the only field IDs there. Don't
spend time hunting for a label; apply the rule and verify the
constructed IDs exist in the listing.

## 9. Global flags (relevant to skills)

| Flag                  | Short | Use                                                                                  |
|-----------------------|-------|--------------------------------------------------------------------------------------|
| `--output`            | `-o`  | `json` (default) / `table` / `csv`                                                   |
| `--fields`            | —     | Select output columns; dot-notation for nested                                       |
| `--flatten`           | —     | Expand nested data                                                                   |
| `--verbose`           | `-v`  | Show request/response details + **request IDs** — gold for troubleshooting           |
| `--quiet`             | `-q`  | Suppress spinners/hints                                                              |
| `--no-color`          | —     | Disable color                                                                        |
| `--profile`           | —     | Named profile (see §3)                                                               |
| `--api-key`           | —     | Override credential                                                                  |
| `--timeout`           | —     | e.g. `30s`, `5m`                                                                     |
| `--no-retry`          | —     | Skip the built-in retry                                                              |

Retry behavior: 3 total attempts, exponential backoff (max delay 20s),
honors `Retry-After` on HTTP 429.

## 10. Exit codes (skill error-handling)

| Code | Meaning            | Skill should…                              |
|------|--------------------|--------------------------------------------|
| 0    | Success            | proceed                                    |
| 1    | General error      | read stderr; default branch                |
| 64   | Usage / bad flags  | the agent's command was malformed — fix it |
| 65   | Auth failure       | suggest re-auth (`supermetrics login`)     |
| 69   | Unavailable        | network / timeout — retry or back off      |

The troubleshooting and validation skills should branch on exit code, not
parse stderr text, wherever possible.

## 11. JSON file inputs

Several commands take complex parameters from files:

```
--connector-file connector.json
--configuration-file config.json
```

This is what makes the per-connector project folder (core knowledge §7,
Phase 3) practical — the agent doesn't have to construct giant inline
arguments, just point the CLI at saved files.

## 12. Gaps to confirm before authoring specific skills

Items not currently in the README that will matter once skills get
implemented:

- **Configuration schema location** — needed for editor schema,
  AI-side checking, and pre-save validation. See §13.
- **Configuration validate / lint** — no `validate` subcommand exists
  today; only `update` enforces correctness. See §13 for the
  workaround.
- **`connector-builder-logs get` response shape** — drives prompt
  design for the troubleshooting skill.
- **Missing-secret runtime behavior** — exit code and error string when
  a referenced secret is absent from the store.

Resolve each by reading `supermetrics <cmd> --help` (and/or the CLI
source) at the moment the relevant skill is being written.

## 13. Validation today (and what we want)

**Today.** The CLI's only validator is `connector-builder update`. It
either succeeds or returns an error. There is no offline schema check,
no `validate`/`lint` subcommand, no dry-run.

**The cost.** Every save is a network round-trip and (because there's
no draft separation, §4) effectively a publish to the Connector. For
an iterative AI agent, that's a slow and visible feedback loop.

### Skill-side strategy (committed)

Two layers of local check before the agent ever calls
`connector-builder update`:

**Layer A — Pre-flight checklist (agent-applied, no script).**

The configuring skill walks through this list before every save. The
agent runs the relevant CLI calls and inspects its own file:

- The Configuration file parses as valid JSON.
- Every required root section is present (auth/connection, secrets,
  accounts where applicable, reporting, response handling).
- Every secret ID referenced from elsewhere in the Configuration is
  declared in the root `secrets` object.
- Every declared secret ID exists in the CLI secrets store
  (cross-checked via `connector-builder-secrets list`).
- Every field ID referenced by a Report Type is declared.
- Cross-references inside the Configuration resolve (e.g. account
  selectors, response paths).

These are instructions for the agent to apply, **not** a script we
maintain. They live in the configuring skill's SKILL.md as a numbered
checklist.

**Layer B — JSON Schema validation (third-party tool, pre-requisite).**

The skill assumes a standard JSON Schema validator is installed and
invokes it against the schema file (see §14). We do **not** ship our
own validator. Verified options:

- **[`check-jsonschema`](https://github.com/python-jsonschema/check-jsonschema)
  — recommended default.** Python, `pipx install check-jsonschema`.
  Works against the Supermetrics schema **out of the box** with no
  flags. Gives readable errors
  (e.g. `Additional properties are not allowed ('name' was unexpected)`).

  ```bash
  check-jsonschema --schemafile "$CACHE" config.json
  ```

- **[`ajv-cli`](https://github.com/ajv-validator/ajv-cli) — works,
  but needs `--strict=false`.** Node, `npm install -g ajv-cli` (or
  on-demand via `npx`). Strict mode rejects the schema because it
  uses a non-standard `docs` keyword.

  ```bash
  ajv validate -s "$CACHE" -d config.json --spec=draft2019 --strict=false
  ```

- Any other Draft 2019-09–compatible validator the user already has,
  if it tolerates unknown keywords.

The configuring skill's SKILL.md lists these as prerequisites at the
top, with install hints, and shows the exact one-liner to run against
`config.json` and the cached schema.

⚠️ **Schema-level validation is necessary but not sufficient.** The
Supermetrics schema is permissive at the root — an empty `{}` passes.
The Layer A pre-flight checklist (required sections present, secret
IDs cross-referenced, etc.) catches the structural mistakes that
schema validation alone misses. Always run Layer A.

### What we still want from upstream

A `supermetrics connector-builder validate --configuration
<string|file>` subcommand would replace Layer B with the
authoritative server-side validator and remove the schema-drift
problem entirely. Tracked as an upstream feature request.

## 14. Schema

The JSON Schema for Connector Configurations is **public and
authoritative**:

- **URL:** https://api.supermetrics.com/connector_builder/v1/schema/v1
- **Dialect:** JSON Schema Draft 2019-09 (`$schema:
  https://json-schema.org/draft/2019-09/schema`)
- **`$id`:** `https://api.supermetrics.com/cc/v1/schema/connector`
- **Title in payload:** _"Supermetrics Hybrid Connector Configuration
  Schema v1"_
- **Size:** ~320 KB
- **Changes over time.** The schema is updated as the platform evolves.
  Do **not** vendor a static copy under `docs/schema/`; that copy will
  silently go stale.

### Cache-and-refresh strategy (session-friendly)

The configuring / validation skills fetch the schema on demand and
reuse it within a session. No bundled copy, no script.

**Cache location:** `~/.cache/supermetrics-connector-builder/connector-schema.v1.json`
(follows XDG conventions; one shared copy across all per-connector
project folders on the machine).

**Refresh rule:**

- If the cached file exists and was modified within the last **24 hours**,
  use it as-is.
- Otherwise, fetch it from the URL above (`curl -fsSL`) and overwrite
  the cache file.

**Pseudo-instruction the configuring skill embeds:**

```
SCHEMA_URL=https://api.supermetrics.com/connector_builder/v1/schema/v1
CACHE=~/.cache/supermetrics-connector-builder/connector-schema.v1.json
mkdir -p "$(dirname "$CACHE")"
if [ ! -f "$CACHE" ] || [ -n "$(find "$CACHE" -mmin +1440 -print 2>/dev/null)" ]; then
  curl -fsSL "$SCHEMA_URL" -o "$CACHE.tmp" && mv "$CACHE.tmp" "$CACHE"
fi
```

The skill SKILL.md states the URL, the cache path, and the 24-hour
TTL in plain prose; the snippet above is the canonical form the agent
runs when it needs the schema. (No new script — the agent emits the
commands directly when it's time to validate.)

### Pointing `config.json` at the schema (editor + agent ergonomics)

The skill should instruct users to add a `$schema` reference to the
top of `config.json`:

```json
{
  "$schema": "https://api.supermetrics.com/connector_builder/v1/schema/v1",
  "info": { … },
  …
}
```

Editors that understand `$schema` (VS Code, JetBrains, etc.) get
autocomplete and inline validation for free. The agent gets a
self-describing file.

### Validation command (Layer B from §13)

Once the cache file is ready, one of:

```bash
# check-jsonschema (Python)
check-jsonschema --schemafile "$CACHE" config.json

# ajv-cli (Node)
ajv validate -s "$CACHE" -d config.json --spec=draft2019
```

Either is acceptable. The skill detects what's installed and falls
back to install instructions if neither is present.

### Schema is a validator, not a vocabulary

The schema's top-level shape (e.g. optional `info` / `metaInformation`)
does not 1:1 match the conceptual root-section list in
`connector-core-knowledge.md §4 / Phase 4` (auth, secrets, accounts,
reporting, response handling). The conceptual model is **authoritative
for skill content**; the schema is used only to mechanically check that
the resulting JSON is well-formed and structurally accepted.

Specifically:

- Skills do **not** use the term **"Hybrid Connector"** even though it
  appears in the schema title. Use **Connector**.
- Skills do **not** surface `info` or `metaInformation` unless the
  user explicitly asks. Both are optional and customer-irrelevant.
- The schema can change shape; the mental model is more stable. When
  they disagree, the configuring skill adapts its **schema bindings**
  to fit the mental-model prose, not the other way around.
