---
name: connector-scoping
description: Use when the user wants to build a new Supermetrics Connector and the scope is not yet defined; or when the user shares an existing integration artifact (vibe-coded Python/Node script, curl, Postman collection, dashboard) to productize; or when the user wants to edit an existing Connector by name.
---

# Phase 1 — Define the scope

Output of this skill: an agreed, written-down **scope statement** for
one Connector, plus an initialized per-Connector project folder.

A scope statement names **the platform** and **the kind of data**.
Examples:

- _"Fetch typical campaign efficiency data from Acme Ads."_
- _"Pull product-level sales data from Beta Marketplace's REST API."_
- _"Scrape a rarely changing JSON dictionary from Gamma's status endpoint."_

A generic goal like _"build a connector to Facebook"_ is **not** a
valid scope — Facebook exposes many APIs with different contracts.
Sharpen until both platform and data kind are explicit.

### Reportability check (apply before accepting any scope)

A Connector produces **Reports** — tabular data composed of
**Dimensions** (categorizing attributes) and **Metrics** (numeric
values). The scope must describe data that can be expressed as a
Report. Reject or reshape scopes that cannot.

Quick test — _"Could a marketer plot this as a chart or pivot it
into a table?"_

| Shape | Reportable? | Handling |
|---|---|---|
| Aggregate / time-series / breakdown — e.g. "clicks by day by campaign" | ✅ Yes | One Report Type. |
| Per-entity list with attributes — e.g. "all campaigns with status and budget" | ✅ Yes | One Report Type (Dimensions only is fine). |
| One row with named numbers — e.g. "current account balance" | ✅ Yes (degenerate) | One Report Type with a single row. |
| Single-entity lookup — e.g. "the user with id=123" or "campaign 456" | ❌ No | Not a Report Type. Tell the user a Connector is the wrong tool for this; suggest the underlying API directly or a different mechanism. |
| Mutation — `POST /something`, `PATCH /thing/id` | ❌ No | Not a Report Type. Connectors are read-only. |
| Auth / session endpoint — `/login`, `/me`, token exchange | ❌ No | This belongs inside the Authentication / Connection identity sections in Phase 2, not Reporting. |
| Account listing | ❌ No | This belongs inside the Accounts section, not Reporting. |

If the user's scope hits a ❌ row, name the mismatch explicitly:
_"The Supermetrics Connector model handles tabular reports
(Dimensions + Metrics). What you've described is a single-entity
lookup, which doesn't fit. Want to back up and aim at the
report-shaped data instead?"_

Do not silently convert a non-report request into a fake Report Type.
That produces a Connector that runs but returns nonsense.

## Detect the entry path

| If the user… | Path | Then… |
|---|---|---|
| Names a platform and roughly what data, with nothing else | **Greenfield** | Elicit scope by asking. |
| Shares a working artifact (script, curl, collection, dashboard) | **Brownfield** | Read the artifact and propose the scope. |
| Names an existing Connector or asks to edit one | **Existing** | List → pick → `get` → seed folder → hand off to Phase 4. |

If multiple apply, ask the user which one they want. Don't assume.

## Greenfield path

Ask the user, in order, the minimum to get a scope:

1. **Which platform?** (full product name; vendor URL if non-obvious)
2. **Which data?** (campaigns, sales, events, dictionary, …)
3. **At what granularity?** (per-day per-campaign? per-row? per-row plus segments?)
4. **For what purpose?** (sanity-check that fetching this data answers the user's actual question)

Stop asking as soon as the answers fit in one scope sentence. Surface
the sentence for confirmation before moving on.

## Brownfield path — vibe-coded integration is the headline case

The competitive differentiator vs the Connector Builder web UI is
turning a working AI-generated or hand-written integration script
into a real Supermetrics Connector. Typical input shapes:

- A one-file Python or Node script (most common).
- A `curl` command or shell snippet.
- A Postman / Insomnia / Bruno collection.
- A small dashboard or reporting project.

Read the artifact, then extract:

1. **Platform / vendor.**
2. **Endpoints** the script calls (paths, query params, headers).
3. **Authentication** method (header name, token field, OAuth flow if
   any, hard-coded keys to factor out).
4. **Fields** the script references in the response (these become
   the Metrics and Dimensions in Phase 2).
5. **Pagination** the script handles, if any.
6. **Sample responses** the script has already received (save these
   later — they pre-pay Phase 3 verification).

Frame the conversion to the user as _"you've proven the integration
works; let's make it production-grade."_ Reassure them that hard-coded
credentials in the script will be moved to the secrets store (Phase 4)
— never copy them anywhere else.

Output: a proposed scope statement + a flagged list of extracted
endpoints / auth / fields. Surface for confirmation; ask only about
gaps.

## Existing-Connector path

The user already has a saved Connector and wants to modify it.

1. Run `supermetrics connector-builder list` to enumerate Connectors
   visible to the user's team.
2. If the user named one, match by name; if multiple match or the
   user didn't name one, present the list and ask.
3. Once chosen, capture its **ds-id** (5-char identifier).
4. Run `supermetrics connector-builder get --connector-identifier <ds-id>`
   to download the current Configuration.
5. Create the project folder (next section) and write the downloaded
   Configuration to `config.json`.
6. Ask the user **what they want to change.** If it's a small fix or
   a new Report Type, hand off to `connector-configuring` (Phase 4)
   directly — the scope is already encoded in the existing config.
   If it's a major scope change (different data, different endpoint
   family), treat it as a fresh greenfield/brownfield from this point.

## Create the per-Connector project folder

Once scope is confirmed (any path), create the working folder under
the user's current working directory:

```bash
mkdir -p <connector-name>/samples <connector-name>/logs
cd <connector-name>
printf '%s\n' '.env' '.team-id' > .gitignore
touch technical-overview.md .env
```

`<connector-name>` should be a slug derived from the scope (e.g.
`acme-campaign-efficiency`). Use kebab-case; keep it short.

If the user is on multiple Supermetrics teams, also write `.team-id`
now — either by asking, or by parsing `supermetrics connector-builder
list` output if there's only one team visible.

## Handoff

- **Greenfield** and **brownfield** → hand off to
  `connector-technical-overview` (Phase 2).
- **Existing** with minor changes → hand off to `connector-configuring`
  (Phase 4) directly. The Tech Overview is implicit in the existing
  config; only re-do Phase 2 if the user is changing scope materially.

## Common mistakes

| Mistake | Fix |
|---|---|
| Accepting a vague scope ("connector for X") and proceeding | Ask "which data?" until the sentence fits a single sentence. |
| Greenfield-eliciting when the user already shared a script | Read the script first; ask only about gaps. |
| Copying secrets from a vibe-coded script into `.env` without flagging | Tell the user explicitly: "these will move to the secrets store in Phase 4; do not commit them." |
| Creating the project folder before scope is confirmed | The folder name is a slug of the scope. Confirm scope first. |

## Deeper reference

- `${CLAUDE_PLUGIN_ROOT}/docs/connector-core-knowledge.md §7 Phase 1` — full workflow context.
- `${CLAUDE_PLUGIN_ROOT}/docs/cli-reference.md §4` — `connector-builder list` / `get` flags.
