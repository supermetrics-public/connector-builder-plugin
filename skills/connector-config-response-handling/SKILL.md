---
name: connector-config-response-handling
description: Use when a Supermetrics Connector Configuration's default response-handling settings (pagination, error handling, shared parsing across Report Types) need to be added, modified, or fixed.
---

# Configure: default response handling

Set or override the Configuration's **default** response-handling
block: pagination, error handling, and any shared parsing rules that
apply across the Connector's Report Types. Per–Report-Type overrides
live in `connector-config-report-type`; this skill owns the defaults.

## Before writing anything — read the docs

```
WebFetch("https://docs.supermetrics.com/docs/error-handling", "<question>")
WebFetch("https://docs.supermetrics.com/docs/request-configuration", "<question>")
```

For **pagination** specifically, the deepest reference lives on the
internal Confluence space (see `${CLAUDE_PLUGIN_ROOT}/docs/docs-portal.md §2`):

```
WebFetch("https://smdocs.atlassian.net/wiki/spaces/CC/pages/395116545/Pagination", "<question>")
```

That page covers every pagination shape Connector Builder supports,
with concrete configuration examples and edge-case handling. Always
read it before authoring the pagination defaults.

If a public-portal slug 404s, update `${CLAUDE_PLUGIN_ROOT}/docs/docs-links.md` and fall
back to Recipe B in `${CLAUDE_PLUGIN_ROOT}/docs/docs-portal.md §2`.

## Pre-conditions

- At least one Report Type has been drafted via
  `connector-config-report-type`, so there's something to share
  defaults across.
- The Tech Overview's **Pagination**, **Rate limits**, and **Errors**
  sections are filled in.

## What to author

### Pagination defaults

Pick the pattern the majority of the Report Types use, encode it
once as the default, then let individual Report Types override.
Patterns the platform supports (see docs):

- Offset / limit.
- Page number (0-indexed or 1-indexed).
- Cursor / next-page-token.
- `Link` header (RFC 5988).
- No pagination (note hard caps on result size).

From the Tech Overview's Pagination section, encode:

- Pattern.
- Param names (and where the cursor / token appears in the
  response).
- Page size — default and maximum.

**Diagnostic discipline.** Pagination is the most common source of
"silently wrong results" — a Connector returns rows but fewer than
it should, because the configuration stops at page one. Before
declaring pagination done:

1. **Confirm the pattern against a real response.** Take the saved
   Phase 3 sample, find the next-page indicator in it (cursor, link
   header, `next` URL, total-pages count, …) and trace where it
   lives. Don't trust documentation alone — many APIs document one
   pagination scheme and use another.
2. **Force at least two pages during Phase 5 validation.** Pick a
   date range or filter that you know returns more rows than one
   page, then verify the row count matches the expected total.
   `--all` on `queries execute` is the single most useful flag for
   this.
3. **Decide explicitly how the configuration knows it's the last
   page.** Empty cursor? Cursor pointing to itself? `null` next URL?
   `has_more: false`? Missing field? Each API expresses
   "no more pages" differently. Document the rule.
4. **Watch for hard result caps** (e.g. "max 10,000 rows regardless
   of pagination"). These belong both here and in the Report Type's
   concerns.

### Error handling defaults

For the **common** error patterns this API returns, map them to
Configuration error-handling primitives. From the Tech Overview's
Errors section:

- Common error shape (HTTP code + body schema).
- Retryable vs terminal classification per status code.
- API behavior on rate-limit hit (429 + `Retry-After`, silent
  throttle, custom code).
- APIs that return errors with 2xx status — explicitly flag and
  handle these (see the `error-handling` docs article).

### Shared response parsing

If multiple Report Types share parsing concerns — e.g. they all wrap
their data in a top-level `{ "data": [...] }` envelope, or all use
the same timestamp format — encode the shared bit here once instead
of repeating in every Report Type.

### Rate-limit awareness

Beyond per-call retry, document any **global** rate-limit budget the
API enforces (e.g. "1000 req/hr per key") so the platform can pace
itself. Per–Report-Type variances belong in
`connector-config-report-type`.

## Verification

After authoring:

1. Schema-validate the full Configuration (Layer B in
   `connector-configuring`).
2. Cross-check each Report Type — for any that override the defaults,
   the override should be intentional and noted in the Tech Overview.
3. Confirm the saved sample in `<project>/samples/` would be parsed
   correctly under the defaults (no Report Type's own response shape
   silently conflicts with the default parsing rule).

## Common mistakes

| Mistake | Fix |
|---|---|
| Setting per-Report-Type pagination on every Report Type instead of a default + selective overrides | Defaults reduce drift. Override only where the API genuinely varies. |
| Ignoring 2xx-error APIs because "they returned 200" | These exist (some marketing platforms in particular). Detect from body shape, not status code. |
| Hard-coding a retry count or backoff in defaults that the CLI already does | Avoid duplicating CLI-level retry. The CLI already retries up to 3× with backoff (`${CLAUDE_PLUGIN_ROOT}/docs/cli-reference.md §9`). |
| Not surfacing rate-limit defaults to the user | Mention global limits in the Tech Overview's Summary so the user knows the operational ceiling. |

## Deeper reference

- `${CLAUDE_PLUGIN_ROOT}/docs/connector-core-knowledge.md §3.3` — Report Type structure.
- `${CLAUDE_PLUGIN_ROOT}/docs/cli-reference.md §9` — CLI-level retry / timeout.
- `${CLAUDE_PLUGIN_ROOT}/docs/docs-links.md` — "Core configuration concepts" section.
