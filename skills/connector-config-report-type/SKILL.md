---
name: connector-config-report-type
description: Use when a Supermetrics Connector Configuration needs one Report Type added or changed — specifically its request shape, response parsing, or field mapping.
---

# Configure: one Report Type

Translate **one** Report Type from the Tech Overview's Reporting
section into the Configuration. Invoke this skill once per Report
Type the Connector exposes — typically several per Connector.

A Report Type is a single integrated blueprint: how to build the
**Request**, how to parse the **Response**, and the resulting
**Fields** (Metrics + Dimensions).

## Before writing anything — read the docs

Two articles cover almost everything you need:

```
WebFetch("https://docs.supermetrics.com/docs/reporting", "<question>")
WebFetch("https://docs.supermetrics.com/docs/request-configuration", "<question>")
```

If the Report Type involves segments, joins, or response reshaping,
also pull from:

```
WebFetch("https://docs.supermetrics.com/docs/joining-data", "<question>")
WebFetch("https://docs.supermetrics.com/docs/transforming-response-structure", "<question>")
```

If a slug 404s, update `${CLAUDE_PLUGIN_ROOT}/docs/docs-links.md` and use Recipe B in
`${CLAUDE_PLUGIN_ROOT}/docs/docs-portal.md §2`.

## Pre-conditions

- `connector-config-auth` and `connector-config-accounts` already
  applied for this Configuration. Report Types depend on both.
- The Tech Overview's Reporting section has a complete sub-section
  for **this** Report Type, including: endpoint + request, response
  structure, example request, example response, field coverage.
- `<project>/samples/<this-report-type>.*` exists and was confirmed
  in Phase 3.

If any of these are missing or ❓, go back to Phase 2 for the gap.

## What to author

For one Report Type, write the Configuration entry covering:

1. **Request shape**
   - HTTP method, URL.
   - Required and optional parameters; which map to **Dimensions**
     vs **Metrics**.
   - Date-filtering parameters and format.
   - Body shape, if applicable (POST / GraphQL).
   - Headers beyond those already set by auth.
2. **Response parsing**
   - JSONPath (or XPath / CSV column expression) to the data rows.
   - For each declared Field: the path inside one row, the type, and
     any required transformation (timestamp parsing, currency
     conversion, etc.).
3. **Field declarations**
   - One entry per Field exposed by this Report Type.
   - Mark each as a Metric (numeric value) or Dimension (categorizing
     attribute).
   - Reference the field IDs from the request-parameter declarations
     if the API requires field-id selection in the request itself.
4. **Per–Report-Type pagination** if the Overview says this endpoint
   paginates differently from the Connector's default response
   handling. Otherwise inherit from
   `connector-config-response-handling`.
5. **Per–Report-Type rate limits** if the Overview noted any. Cross-
   reference, don't duplicate, the global rate-limit notes.

## Verification

After authoring this Report Type's Configuration entry:

1. Schema-validate the whole Configuration (Layer B).
2. Cross-check every Field ID in the parsing block against the saved
   sample in `<project>/samples/`. Every path should resolve to a
   value in the sample.
3. Cross-check every Field used in the request parameter list
   against the field-coverage table in the Tech Overview. No "phantom"
   fields.
4. Hand back to `connector-configuring` for the next Report Type or
   the save.

## Repeating for additional Report Types

Each additional Report Type is a fresh invocation of this skill.
Share **defaults** via `connector-config-response-handling` where
they're identical; only override per–Report-Type where they genuinely
differ. Duplication invites drift.

## Common mistakes

| Mistake | Fix |
|---|---|
| Inventing a field path that "should" exist in the response | Open the saved sample and trace it. If it's not there, mark the Field ❌ in the Tech Overview and don't include it here. |
| Mixing parameter formats from different examples in the same Report Type | Pick one shape from the docs and stay with it. Mixing breaks signing and pagination. |
| Hard-coding a date or account ID into the request because "we're testing" | The Configuration is a template — placeholders / template syntax exist for these. See `${CLAUDE_PLUGIN_ROOT}/docs/docs-links.md` → Placeholders. |
| Forgetting to handle the "no data" empty-response case | Confirm the response-parsing path tolerates an empty rows array. Add it to `connector-config-response-handling` if not. |
| Marking a Field ⚠️ "needs join" and then never joining it | Either implement the join (see `joining-data` doc) or remove the Field. Don't ship phantom Fields. |

## Deeper reference

- `${CLAUDE_PLUGIN_ROOT}/docs/connector-core-knowledge.md §3.3` — Report Types concept.
- `${CLAUDE_PLUGIN_ROOT}/docs/docs-links.md` — "Core configuration concepts" and "Advanced configurations" sections.
