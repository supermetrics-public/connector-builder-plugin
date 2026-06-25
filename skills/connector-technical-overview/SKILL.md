---
name: connector-technical-overview
description: Use when a Supermetrics Connector's scope is fixed but its Technical Overview document does not yet exist or lacks verified endpoint examples.
---

# Phases 2 & 3 — Research and verify

Outputs:

1. **`<project>/technical-overview.md`** — filled-in copy of
   `${CLAUDE_PLUGIN_ROOT}/skills/connector-technical-overview/templates/technical-overview.template.md`, every required section
   present, status emoji ✅⚠️❓❌ on each section.
2. **`<project>/samples/`** — raw request/response captures from real
   calls. At least one sample per Report Type.
3. User has **visually confirmed** at least one sample shows the
   right data.

This is the single highest-leverage artifact in the workflow. A good
Tech Overview makes Phase 4 mostly mechanical translation; a bad one
guarantees rework.

## Sequence

1. **Copy the template.** Read the file at
   `${CLAUDE_PLUGIN_ROOT}/skills/connector-technical-overview/templates/technical-overview.template.md`
   and write a working copy to `<project>/technical-overview.md`.
   Fill the Header section from the confirmed scope.
2. **Read the relevant Supermetrics docs first.** Before touching the
   third-party API, skim what the Connector Builder platform supports
   for each section. Use the curated index in `${CLAUDE_PLUGIN_ROOT}/docs/docs-links.md`
   — at minimum: `authentication`, `accounts`, `reporting`,
   `request-configuration`, `error-handling`, `data-source-user`,
   `supported-apis-and-technologies`. This determines whether the
   target API is buildable at all.
3. **Research the Data Source's API.** Read the vendor's documentation
   for each Report Type's endpoint, the auth method, pagination,
   errors, and rate limits. `WebFetch` and `WebSearch` are your tools
   — `${CLAUDE_PLUGIN_ROOT}/docs/docs-portal.md` recipes apply equally to vendor docs.
4. **Fill the Tech Overview sections in order.** Per section, write:
   docs links, how it works, implementation notes, example
   request, example response, concerns. Mark each section with a
   status emoji. Stay concise; if a section is straightforward, two
   bullets is enough.
5. **Verify with real requests** (this is Phase 3). For each Report
   Type — and for auth + account-listing if applicable — actually
   call the endpoint with `curl` (or whatever the user prefers).
   Save the raw response to `<project>/samples/<descriptive-name>.json`
   (or `.xml`, `.csv`). One sample per endpoint family minimum.
6. **Present a verification sample to the user.** Pick a
   representative Report Type sample, show the user the first
   N rows, and **ask: "is this the data you wanted?"** Wait for
   explicit confirmation before declaring Phase 3 done.
7. **Write the Summary section.** Verdict (Feasible / Feasible with
   caveats / Not feasible), what works, caveats, known risks, and
   field-coverage counts.

## Special handling: brownfield handoff

If we arrived from `connector-scoping` via the **brownfield** path,
much of the Overview is pre-paid:

- Endpoints, fields, auth method came out of the user's artifact.
- Real responses the user's script has already received become
  Phase 3 samples — save them directly to `<project>/samples/`.
- Phase 3 verification mostly becomes "confirm the samples match
  what we expect." Still ask the user explicitly.

Skip nothing — write every required section even if the answer is
"yes this works, see sample X." Auditors of the resulting Connector
need the Overview to be self-contained.

## Probing the API — discipline

When you need to actually call the vendor's API (Phase 3
verification, or to confirm a single ambiguous detail in Phase 2),
follow these rules. They keep the user from being asked to approve
the same kind of command 5 times.

### Tool choice

| Situation | Tool |
|---|---|
| Reading vendor docs (HTML / markdown pages) | `WebFetch` — domain-allowlistable, no Bash, no permission prompt per call. |
| Unauthenticated GET against an API | `WebFetch`, when the response is what matters more than the headers. |
| Authenticated GET / POST / custom-method against an API | `curl` — `WebFetch` does not set request headers. |

### When using `curl`

Two rules — both about reducing permission noise:

1. **One curl per attempt, never a `for` loop.** Loops with shell
   variable expansion (`$q`, `$TOKEN`) trigger Claude Code's
   `simple_expansion` permission prompt on every iteration. The user
   sees the same prompt N times. Write each request as its own
   command, run it, learn from the result, then write the next one.
2. **Read the docs before guessing.** If you find yourself about to
   write a loop probing 5 candidate parameter names (`start_date`,
   `from`, `since`, …), stop. The vendor's docs almost certainly
   specify the correct one — go read them first via `WebFetch`. Blind
   probing wastes attempts, generates permission noise, and risks
   logging in the vendor's audit trail for nothing.

### Secret handling in curl

The Phase 2 working folder has the user's third-party credentials in
`.env`. Two options for passing them to `curl`, each with a tradeoff:

- **`set -a && . ./.env && set +a && curl -H "Authorization: $TOKEN" …`**
  — keeps the secret out of chat history, but triggers
  `simple_expansion` once per command.
- **Read `.env`, paste the literal value into the `curl` command** —
  no shell expansion, no permission prompt, but the secret appears in
  chat history. Acceptable for a customer's third-party API key
  during dev (they can rotate); never acceptable for a Supermetrics
  API key or any production credential.

Default to the first option. The permission prompt is unavoidable;
keep the number of prompts low by following rule #1.

### Why this matters

`simple_expansion` is an open Claude Code limitation
(<https://github.com/anthropics/claude-code/issues/43713>) — there's
currently no way to pre-allow these prompts in `settings.json`. The
skill cannot make them go away, only minimize them.

## Authentication research

Authentication is the section that most often surfaces feasibility
blockers. Always answer these before declaring the auth section ✅:

- Which auth flow does the API use? (API key / OAuth2 grant type /
  OAuth1 / Basic / JWT / Custom)
- Where does the token live (header name, query param, body)?
- Token refresh: required? lifetime? endpoint URL?
- Required scopes / permissions?
- Does the user have to register an app, IP-allowlist, or pre-approve
  anything?

If the answer to any of these is "unclear" or "I think so," the
section is ❓, not ✅. The configuring phase will burn cycles trying
to guess.

## When to stop

Stop when **all four** are true:

1. Every required template section has content and a status emoji.
2. `<project>/samples/` has at least one verified sample per Report
   Type.
3. The Summary section has a verdict and a field-coverage table.
4. The user has visually confirmed at least one sample.

Only then hand off to `connector-configuring`.

## Common mistakes

| Mistake | Fix |
|---|---|
| Skipping the Supermetrics docs step and inventing what the platform supports | Always read the relevant docs-links entries first. The platform's capabilities bound the design space. |
| Marking sections ✅ when they're really ❓ | If you guessed, it's ❓. Be honest — Phase 4 will catch the lies, expensively. |
| Filling sections from the model's training memory instead of fresh fetches | Always `WebFetch` or `WebSearch` for vendor docs. Memorized API details are routinely out of date. |
| Writing the Overview without taking real samples | A Tech Overview without sample responses is a guess. Always make the calls. |
| Handing off without user confirmation | One direct question — "is this the data you wanted?" — before moving on. |

## Deeper reference

- `${CLAUDE_PLUGIN_ROOT}/docs/connector-core-knowledge.md §7 Phases 2–3` — workflow context.
- `${CLAUDE_PLUGIN_ROOT}/docs/docs-portal.md` + `${CLAUDE_PLUGIN_ROOT}/docs/docs-links.md` — looking up
  Supermetrics platform docs.
- `${CLAUDE_PLUGIN_ROOT}/skills/connector-technical-overview/templates/technical-overview.template.md` — the canonical template.
