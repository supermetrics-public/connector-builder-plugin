---
name: connector-validation
description: Use when a Supermetrics Connector Configuration has been saved via supermetrics connector-builder create/update and has not yet been verified by running it against the source.
---

# Phase 5 — Validate

Run the saved Connector through the CLI and confirm it returns the
right data. Validation has two layers; both must pass before the
Connector is declared done:

1. **Doesn't blow up.** `queries execute` returns exit 0 with rows.
2. **Returns the right data.** Output matches the verified samples
   from Phase 3 (within reason).

## Pre-conditions

- The Connector was successfully saved (`connector-configuring`
  succeeded).
- A `ds-id` is captured in `<project>/.ds-id` (or known from the
  `connector-builder create` output).
- Every referenced secret is registered in the secrets store.
- `<project>/samples/` has at least one verified sample per Report
  Type from Phase 3.

## Resolve identifiers before the first CLI call

`Read` these three small files **once** at the start of the skill
and remember the values:

- `<project>/.ds-id` → the Connector identifier (5-char string).
- `<project>/.team-id` → the team scope (number).
- `<project>/.ds-user` → the Login (Connection) username, after
  Step 0 has populated it.

All bash commands below use placeholders (`<ds-id>`, `<team-id>`,
`<ds-user>`) — substitute the **literal** values from those reads
when running each command. **Do not write `"$(cat .ds-id)"` into the
shell** — it triggers a Claude Code permission prompt on every
invocation. These IDs aren't secrets; using literals is correct.

## Sequence

### 0. Ensure a Login (Connection) exists for this Connector

`queries execute` requires not just a saved Connector but also an
authenticated **Login** — the Data Source-side credential (what
core knowledge §5.1 calls a Connection). Without one, every query
returns an auth failure regardless of how clean the Configuration
is.

First, check whether a usable Login already exists. Use a projected
table view — we only need the identifying fields, not the full record:

```bash
mkdir -p logs
supermetrics logins list \
  --fields login_id,display_name,ds_info.ds_id,username \
  > logs/logins.txt 2>&1
```

Read `logs/logins.txt` and find a row whose `DS_INFO.DS_ID` matches
the current `<ds-id>`. If one exists, capture its `USERNAME` (this
is what `queries execute --ds-user` accepts — not the `LOGIN_ID`)
into `<project>/.ds-user` and skip to step 1.

If none exists, create one:

```bash
supermetrics login-links create \
  --ds-id <ds-id> \
  --description "Login for <connector-name> validation" \
  --expiry-time "+24h" \
  --output json \
  > logs/login-link.json 2>&1
```

The output contains a URL. The skill must then:

1. **Surface the URL to the user, explicitly.** Format it as a
   plain link they can click — do not hide it inside a JSON dump.
2. **Tell the user what to do**: open the URL, complete the
   authentication on the vendor's site, return to the terminal.
3. **Wait for confirmation.** Ask the user to reply when the
   authentication is done. Do not poll the API in a tight loop.
4. **Verify the Login appeared:**

   ```bash
   supermetrics logins list \
     --fields login_id,display_name,ds_info.ds_id,username \
     > logs/logins.txt 2>&1
   ```

   Find the new Login matching the current `<ds-id>`, capture its
   `USERNAME` (this is what `queries execute --ds-user` accepts) into
   `<project>/.ds-user`.
5. **Close the used link** so the URL can't be reused:

   ```bash
   supermetrics login-links close --link-id <link-id> > logs/login-link-close.json 2>&1
   ```

   (Flag names per `--help`.)

If a previously-cached `.ds-user` exists but `logins list` no longer
shows it (expired, revoked, deleted), repeat the create-link flow
above.

### 1. Discover the runtime field IDs

**Important:** the field IDs declared in `config.json` are **not** the
field IDs `queries execute --fields` expects. At runtime they get
prefixed with the Report Type ID.

**Schema v1 rule:**

```
runtime_field_id = <report_type_id> + "_" + <config_field_id>
```

Example: a field declared as `clicks` inside Report Type `ad_perf` is
queried as `ad_perf_clicks`. Underscore separator, not dot.

**Construct runtime IDs from the Configuration directly using this
rule.** Do not try to scan `datasource get` for a "prefixed field
ID" label — there isn't one. The prefixed IDs do appear in the
`datasource get` field listing, but they're not annotated. Treat
that listing as the cross-check, not the discovery mechanism.

```bash
mkdir -p logs
supermetrics datasource get \
  --data-source-id <ds-id> \
  --team-id <team-id> \
  --output json \
  > logs/datasource.json 2>&1
```

Verification: for each field ID you constructed, confirm it exists
in `logs/datasource.json`. If one doesn't, either:

- The Configuration is wrong (field not declared, declared under a
  different Report Type, or has a typo) — fix in Phase 4.
- The schema version is no longer v1 and the prefix rule has changed
  — adapt this skill.

Pass the **runtime** IDs to `queries execute --fields`, never the
raw Configuration IDs.

### 2. Pick one Report Type to validate first

Start with the simplest Report Type — the one with the fewest fields
and no segments / joins. If everything works, expand to others. If
nothing works, the simplest one is the easiest to debug.

### 3. (If needed) discover real Account IDs

If you don't already have a known account ID and the user hasn't
specified one:

```bash
supermetrics accounts list \
  --ds-id <ds-id> \
  --ds-users <ds-user> \
  > logs/accounts.txt 2>&1
```

Read `logs/accounts.txt` to find the account ID column (the column
name will vary by Data Source — common names are `ID`, `ACCOUNT_ID`,
or it may be nested as `ACCOUNTS.ID` and need `--flatten`). Pick one
account ID for the first validation run; expand later.

### 4. Run `queries execute`

Use the **runtime** field IDs (from step 1), not the Configuration's
field IDs:

```bash
supermetrics queries execute \
  --ds-id <ds-id> \
  --ds-user <ds-user> \
  --fields <runtime-field-ids-comma-separated> \
  --start-date <YYYY-MM-DD> \
  --end-date <YYYY-MM-DD> \
  --ds-accounts <real-account-id> \
  --limit 10 \
  --output json \
  > logs/queries-execute.json 2>&1
```

The Bash tool's own return tells you the exit code — no need to
`echo "exit=$?" > .exit`. Read `logs/queries-execute.json` for the
body (success rows or error envelope).

Notes:

- Use the same date range that produced the saved sample, if you
  have it. Otherwise pick a small recent window.
- `--limit 10` keeps the first call cheap; expand only after the
  shape matches.
- `--output json` gives parseable output for comparison.
- Always **redirect** with `> file 2>&1`. **Do not pipe to `tee`** —
  it triggers a Claude Code permission prompt on every run and the
  prompt is unnecessary. Read the file with `Read` afterwards if you
  need to see the content.
- **For pagination verification**, do **not** use `--all` — against a
  Connector Builder source it can stop after the first page. Use
  `--max-rows <N>` (e.g. `--max-rows 1000`) to force the platform
  through the configured pagination until N rows or the API runs out.
  See `${CLAUDE_PLUGIN_ROOT}/docs/cli-reference.md §8` for the full
  rationale.

### 5. Branch on exit code

The Bash tool returns the exit code of the `queries execute`
invocation directly — read it from the tool's result, not from a
separate `.exit` file.

| Exit | Meaning | Action |
|---|---|---|
| 0 | Success — rows returned | Go to step 6. |
| 65 | Auth failure | Hand off to `connector-troubleshooting`. **First** suspect a missing or wrong secret, not OAuth itself. |
| 64 | Usage / bad flags | The CLI command is malformed. Fix the flags. Common cause: field IDs from `config.json` were passed instead of the runtime IDs from `datasource get` (step 1). |
| 69 | Network / timeout | Retry once; if it persists, hand off to `connector-troubleshooting`. |
| 1 | Generic error | Read `logs/queries-execute.json` for stderr; hand off to `connector-troubleshooting`. |

### 6. Compare to the Phase 3 sample

Compare the CLI's JSON output against
`<project>/samples/<this-report-type>.*`:

- **Schema match**: every Field declared in the Report Type is
  present in every row.
- **Value spot-check**: pick 2–3 rows from the sample and find the
  same rows in the CLI output (same key dimensions, same metric
  values within rounding).
- **Cardinality plausibility**: roughly the same number of rows
  for the same date window. Drift > ~20% is a flag.

What counts as a real mismatch:

- A Field is missing entirely or `null` for every row (parsing bug).
- A Metric value is off by a factor (units / currency / timezone bug).
- The CLI returns rows that aren't in the sample, or vice versa,
  unrelated to date-range effects.

What does **not** count:

- Order differences (the platform may sort differently).
- Small per-day cardinality variance (the source may backfill / change
  data day-to-day).
- Extra system-added Fields the platform injects.

### 7. Expand or hand off

- **All Fields match** → repeat for the next Report Type.
- **All Report Types match** → declare the Connector validated.
  Optional: `connector-logo` to upload a logo.
- **Anything mismatches** → hand off to `connector-troubleshooting`
  with the diff.

## Logs and cleanup

The `logs/` folder fills up fast during iteration. Keep it clean:

- All CLI invocations in this skill redirect to `logs/` with
  `> logs/<name>.<ext> 2>&1` (no `tee`, no chains — the Bash tool
  surfaces the exit code on its own; `Read` the file afterwards for
  content). **Never use `tee`** — it triggers a Claude Code
  permission prompt on every run and adds no value beyond `>` +
  `Read`. **Don't chain commands with `;`** either — same kind of
  prompt.
- Treat `logs/` as scratch space. Files there are diagnostic, not
  authoritative — never rely on a `logs/` file across phases.
- **After a successful validation**, prune `logs/` to just the
  most-recent run per Report Type, or wipe it entirely. Leaving
  hundreds of stale outputs in the project folder clutters the
  context for future sessions.

Useful files to *briefly* keep around during a session:

- `logs/datasource.json` — the runtime view; reused across multiple
  validation runs.
- `logs/logins.txt`, `logs/accounts.txt` — projected tables; reused.
- `logs/queries-execute-<report>.json` + `.exit` — the latest
  validation run per Report Type.

If a Report Type's last run was successful, its `queries-execute`
log can be deleted at the next "all green" checkpoint.

## CLI output style

When invoking the CLI in this skill, follow the rule from
`${CLAUDE_PLUGIN_ROOT}/docs/cli-reference.md §9a`:

- **Specific field(s) from output** → `--fields <path>[,…]`. Default
  output is already table; no `--output table` needed. Use
  lowercase JSON keys with dot-notation for nesting.
- **Nested array shows as `foo: N items`** → add `--flatten` (or
  `--output csv`) to expand it into rows.
- **Empty table when you expected data** → wrong field name. Run
  once without `--fields` to see the actual keys, then fix.
- **Need to skim the response** → `--output json > logs/<name>.json`,
  then `Read` the file.
- **Real compute** (count, filter, transform) → Python or `jq`.

This skill's examples follow that pattern: projected tables for
discovery (`logins list`, `accounts list`), full JSON storage only
for `datasource get` (we scan it for runtime IDs) and
`queries execute` (we diff it against Phase 3 samples).

## Common mistakes

| Mistake | Fix |
|---|---|
| Validating the most complex Report Type first | Start simple. Complex Report Types compound failure modes. |
| Declaring success on exit 0 without comparing data | Exit 0 means the CLI ran, not that the data is right. Always diff against the sample. |
| Calling the field-coverage diff a "mismatch" when it's just sort order | Compare by key Dimensions, not by row index. |
| Re-running with bigger `--limit` after the first failure | Reduce noise, not increase it. Fix the small failure first. |
| Assuming exit 65 is an OAuth problem | Most exit-65 cases in this workflow are missing secrets. Check `connector-builder-secrets list` first. |

## Deeper reference

- `${CLAUDE_PLUGIN_ROOT}/docs/connector-core-knowledge.md §7 Phase 5` — workflow context.
- `${CLAUDE_PLUGIN_ROOT}/docs/cli-reference.md §§8, 10` — `queries execute` syntax and exit codes.
