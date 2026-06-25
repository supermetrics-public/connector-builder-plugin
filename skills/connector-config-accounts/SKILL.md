---
name: connector-config-accounts
description: Use when a Supermetrics Connector Configuration's Accounts section (listing Accounts under a Connection, or selecting which Accounts to fetch) needs to be added, modified, or fixed.
---

# Configure: Accounts

Translate the Tech Overview's **Accounts** section into the
Configuration's Accounts block.

An Account is an independent scope of data under one Connection
(e.g. one of many ad accounts the user has access to). The Connector
needs to know how to list them and which Account attributes flow
into later requests.

## Before writing anything — read the docs

```
WebFetch("https://docs.supermetrics.com/docs/accounts", "<question>")
```

If the Tech Overview indicates the source has no Account concept
("single-account" / "connection-as-account"), also confirm how to
express that in the Configuration — the same article covers both.

If the slug 404s, update `${CLAUDE_PLUGIN_ROOT}/docs/docs-links.md` and use Recipe B in
`${CLAUDE_PLUGIN_ROOT}/docs/docs-portal.md §2`.

## Pre-conditions

- `connector-config-auth` has already been applied for this
  Configuration. Account listing typically uses the authenticated
  Connection; without auth it can't be tested.
- The Tech Overview's **Accounts** section is filled in with: account-
  listing endpoint URL, hierarchy (if any), per-Account fields, and
  pagination notes.

If those are missing or ❓, go back to Phase 2.

## What to author

From the Tech Overview, capture into the Configuration:

1. **Account-listing request** — the endpoint to call to enumerate
   Accounts available to the Connection. Often a `/accounts`,
   `/properties`, `/workspaces` endpoint; sometimes a `/me` payload
   includes them.
2. **Response parsing** — where in the response the account list
   lives (JSONPath), and the per-Account fields to extract:
   - **Account ID** (required) — what later requests reference.
   - **Account name** (typically required for UI display).
   - Optional fields: currency, timezone, hierarchy parent, status,
     access level.
3. **Pagination on the listing** — if applicable. Pattern (cursor /
   page / offset) should match what the Overview's Pagination
   section recorded.
4. **Selection attributes** — which Account fields are usable in
   `queries execute --ds-accounts` selectors. At minimum the ID.

For single-account APIs (no Account concept on the source side),
encode the "connection-as-account" pattern as described in the docs
article — usually a constant single Account derived from the
Connection identity.

## After writing — verify

1. The Account-listing request would actually return data using the
   Connection credentials (cross-check with the verified sample from
   Phase 3 if available; otherwise call it yourself with `curl`).
2. The selected fields exist in the real response.
3. Hand back to `connector-configuring` for save and the next
   sub-task.

## Common mistakes

| Mistake | Fix |
|---|---|
| Picking an Account ID field that's nested or computed | Use the simplest stable identifier the API exposes. Document why if you had to derive it. |
| Skipping pagination on the listing because "the user only has 3 accounts" | Customers move to larger teams. If the API paginates, configure pagination. |
| Treating single-account sources as multi-account anyway | Use the "connection-as-account" pattern from the docs; don't fake an Account list. |
| Including PII / unused metadata in the per-Account fields | Capture only what later Report Types or the UI actually use. |

## Deeper reference

- `${CLAUDE_PLUGIN_ROOT}/docs/connector-core-knowledge.md §5.2` — Accounts concept.
- `${CLAUDE_PLUGIN_ROOT}/docs/docs-links.md` — "Core configuration concepts" section.
