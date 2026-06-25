---
name: connector-troubleshooting
description: "Use when a Supermetrics Connector behaves wrong: validation mismatch, missing or extra data, runtime error, auth failure (exit 65), unexpected fields, or empty results — including standalone cases where the user lands here without prior workflow context."
---

# Troubleshoot a misbehaving Connector

Two entry contexts:

- **In-workflow** — `connector-validation` (Phase 5) handed off
  because a save succeeded but the output is wrong.
- **Standalone** — the user landed here directly: _"my Acme
  Connector started returning weird data this morning"_, with no
  prior workflow context.

In both cases, the goal is the same: identify the root cause without
re-doing whole phases of work.

## First — collect the inputs

If standalone, before diagnosing anything:

1. **Connector identifier (`ds-id`)** — ask the user or list:

   ```bash
   supermetrics connector-builder list --team-id <team-id>
   ```

2. **Current Configuration** —

   ```bash
   supermetrics connector-builder get \
     --team-id <team-id> \
     --connector-identifier <ds-id>
   ```

3. **Symptom** in the user's words: error message, missing data,
   wrong values, total silence.

4. **Last-known-good state** if any (date / change / version).

## Diagnostic ladder

Walk in order. Stop at the first match.

### Rung 1 — Did the CLI command itself fail? Branch on exit code

| Exit | Almost always means | Next step |
|---|---|---|
| 64 | The agent's command is malformed | Re-form the command per `${CLAUDE_PLUGIN_ROOT}/docs/cli-reference.md`. |
| 65 | **Likely a missing secret**, not OAuth (see Rung 2) | Rung 2. |
| 69 | Network / timeout transient | Retry once with `--timeout 60s`. Persistent → check Supermetrics status page or vendor status page. |
| 1 | Generic — read stderr | Rung 3+. |
| 0 | Command ran; the **data** is wrong | Rung 4+. |

### Rung 2 — On exit 65, check three things in order

Exit 65 (auth failure) has at least three indistinguishable causes
in this workflow. Walk this list before assuming the worst:

**2a. Missing secret.** A Configuration that references a secret ID
not registered in the store returns a generic auth failure with **no
naming of the missing secret**. Check:

```bash
supermetrics connector-builder-secrets list \
  --team-id <team-id> \
  --connector-identifier <ds-id> \
  --output json > logs/cb-secrets.json 2>&1
```

Cross-reference against the Configuration's root `secrets` object:

- Every ID declared in `secrets` should appear in the `list` output.
- Every secret referenced inside auth / Connection / request
  definitions should be declared in `secrets`.

If a gap exists, register the missing one(s) via
`connector-builder-secrets create` (see `connector-config-auth`).
Re-run the failing call.

**2b. Missing or expired Login (Connection).** Even with all secrets
present, `queries execute` fails if there's no authenticated Login
for this Connector's Data Source. Check:

```bash
supermetrics logins list --output json > logs/logins.json 2>&1
```

Look for an entry matching the current `<ds-id>` that is not
expired. If none exists, hand off to `connector-validation` step 0
(or `connector-config-auth` if the user is still in Phase 4) to
create one via `login-links create`.

**2c. Actual auth-token problem.** Only if both 2a and 2b are clean,
treat exit 65 as a real token issue (expired refresh token, scope
drift, IP allowlist change, vendor-side policy change). Read the
CLI's request/response logs (Rung 3) for the underlying response.

### Rung 3 — Read the CLI's request/response logs

```bash
mkdir -p logs

supermetrics connector-builder-logs list \
  --team-id <team-id> \
  --connector-identifier <ds-id> \
  --output json \
  > logs/cb-logs-list.json 2>&1

supermetrics connector-builder-logs get \
  --team-id <team-id> \
  --connector-identifier <ds-id> \
  --log-id <id> \
  --output json \
  > logs/cb-log-<id>.json 2>&1
```

(Flag names per `--help`; defaults may vary.) Use `--verbose` on the
original failing command to capture request IDs that correlate with
log entries.

**Always redirect with `> file 2>&1`. Never use `tee`** — it triggers
a Claude Code permission prompt on every run and adds no value over
`>` + a follow-up `Read`. Apply this rule to every CLI invocation in
this skill.

What to look for in the logs:

- The actual outbound request the platform built — including the
  resolved auth header, params, body. Compare to the Tech Overview's
  example request.
- The raw response from the source. Compare to the Phase 3 sample if
  one exists.
- Any retries, the exact response that triggered them, and the final
  outcome.

### Rung 4 — Field / parsing drift

When exit is 0 but the data is wrong, the failure mode is almost
always one of:

| Symptom | Likely cause | Fix |
|---|---|---|
| All rows have `null` for one Field | Wrong JSONPath / response path | Re-derive path from a fresh sample. |
| Values off by a factor of 100 (or 1/100) | Currency in cents/units, not currency in dollars | Add a transform in `connector-config-report-type`. |
| Off-by-one days | Timezone or inclusive/exclusive end date | Compare to Tech Overview's Date/Timezone section. |
| Far fewer rows than the sample | Pagination not consuming all pages | Check pagination config in `connector-config-response-handling`. |
| Extra rows that aren't in the sample | Sample was filtered; or the sample is stale | Refresh the sample, re-compare. |
| Whole Report Type empty since a known date | API changed shape, was deprecated, or the user lost access | Re-run Phase 2 research for that endpoint; ping vendor status. |

### Rung 5 — Configuration drift from the Tech Overview

If Rungs 1–4 don't surface the cause:

1. Re-read the Tech Overview's section for the affected area.
2. Diff against the relevant Configuration section.
3. If the Configuration matches the Overview and behavior is still
   wrong, the Overview itself is stale — re-do Phase 2/3 for that
   area, then come back here.

## Redaction discipline

Never echo secret values to the user, to chat output, to commit
messages, or to log files. When pasting request examples into chat
during a diagnosis, redact:

- Authorization headers.
- API keys in query params.
- OAuth tokens in bodies.
- Any field the Configuration's `secrets` object references.

Replace with `<redacted>`. If the user shows you a token by mistake,
recommend rotating it via `connector-builder-secrets update`.

## Handoff

- **Fix is a Configuration change** → hand off to
  `connector-configuring` (or directly to the affected
  `connector-config-*` sub-skill).
- **Fix requires re-research** (API changed, scope changed, fields
  missing) → hand off to `connector-technical-overview`.
- **Fix is environmental** (creds rotated, IP allowlist, vendor
  status) → guide the user; nothing to author.
- **Resolved** → re-run `connector-validation`. Don't declare done
  until the validation passes.

## Common mistakes

| Mistake | Fix |
|---|---|
| Treating every exit 65 as an OAuth issue | Rung 2 first. Most are missing secrets. |
| Re-doing Phase 2 before reading the logs | Logs almost always reveal the cause in minutes. Re-research is a last resort. |
| Patching a Configuration directly without updating the Tech Overview | Whatever was wrong in the Overview will rot back into the Configuration next time. Fix both. |
| Echoing a token into the chat while explaining the issue | Redact. Recommend rotation if the user already echoed one. |

## Deeper reference

- `${CLAUDE_PLUGIN_ROOT}/docs/cli-reference.md §6` — `connector-builder-logs` commands.
- `${CLAUDE_PLUGIN_ROOT}/docs/cli-reference.md §10` — exit-code semantics.
- `${CLAUDE_PLUGIN_ROOT}/docs/connector-core-knowledge.md §§5–7` — concepts being diagnosed.
