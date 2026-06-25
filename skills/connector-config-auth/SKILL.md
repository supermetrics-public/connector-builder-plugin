---
name: connector-config-auth
description: Use when a Supermetrics Connector Configuration's authentication or Connection section needs to be added, modified, or fixed.
---

# Configure: Authentication, Connection, and Secrets

Translate the **Authentication** and **Connection identity** sections
of the Technical Overview into the corresponding Configuration
sections, plus the root `secrets` object.

This is the hardest single area of a Configuration. Most missing-
field and runtime errors trace back to here.

## Before writing anything — read the docs

The exact JSON structure for each auth flavor lives in the docs
portal. Always fetch fresh; do not rely on memorized examples.

1. Start with the auth overview:
   `WebFetch("https://docs.supermetrics.com/docs/authentication", "<question>")`.
2. Pick the auth-flavor article that matches the Tech Overview:
   - **API key in header / query param** →
     `https://docs.supermetrics.com/docs/api-key-authentication`
   - **OAuth2** →
     `https://docs.supermetrics.com/docs/oauth2-authentication`
   - **OAuth1** →
     `https://docs.supermetrics.com/docs/oauth1-authentication`
3. Read the Connection-identity reference:
   `https://docs.supermetrics.com/docs/data-source-user`
4. Read the secrets reference (covers the root `secrets` object's
   exact shape):
   `https://docs.supermetrics.com/docs/secrets`
5. If signing is required, also:
   `https://docs.supermetrics.com/docs/request-signers`

If `WebFetch` returns a 404 for any of these, update the slug in
`${CLAUDE_PLUGIN_ROOT}/docs/docs-links.md` and fall back to the Recipe B search in
`${CLAUDE_PLUGIN_ROOT}/docs/docs-portal.md §2`.

## Translate from the Tech Overview

From the Overview's **Authentication** section, you should already have:

- The auth flavor.
- Token endpoint URL and request body format (for OAuth-style flows).
- Header / param name carrying the credential on data requests.
- Token expiry and whether refresh is needed.
- Required scopes / permissions.
- User-side setup steps (creating an app, etc.).

From the Overview's **Connection identity (Data Source User)** section:

- Whether a `/me`-style endpoint exists; if so, its URL.
- Fields to persist alongside the token (user id, account id, etc.).
- How the access token is referenced in subsequent data requests.

If any of these are missing or marked ❓ in the Overview, **stop and
go back to Phase 2**. Do not guess.

## Wire secrets

Every credential value goes through the secrets store. For each
secret the Configuration needs:

1. Declare its ID in the root `secrets` object (shape per the docs
   article above).
2. Reference the ID from the Authentication / Connection sections.
3. Register the value via the CLI — interactively, masked:

   ```bash
   supermetrics connector-builder-secrets create \
     --team-id "$(cat .team-id)" \
     --connector-identifier <ds-id> \
     --secret-name <id>
   # CLI prompts for the value with masked input
   ```

   (If `<project>/.env` has the value handy, the user pastes it in;
   the file `.env` itself is never auto-loaded.)

After registration, confirm presence:

```bash
supermetrics connector-builder-secrets list \
  --team-id "$(cat .team-id)" \
  --connector-identifier <ds-id>
```

A missing secret will surface at runtime as a generic auth failure
(exit 65) with no naming — see `connector-troubleshooting`.

## OAuth2 specifics

OAuth2 is where most users get stuck. Confirm before coding:

- **Grant type**: client credentials, authorization code, …
- **Token endpoint**: full URL, expected body shape (JSON vs
  form-encoded).
- **Token response**: which field holds the access token, which holds
  the refresh token, which holds the expiry.
- **Header injection**: which header name on data requests
  (`Authorization: Bearer …`? `X-Vendor-Token: …`?).
- **Scopes**: comma-separated, space-separated, or repeated param?

Mirror the docs examples exactly for the chosen grant type. Don't
mix and match patterns from different examples.

## After writing — verify

1. The Configuration parses.
2. Every secret ID referenced is declared in `secrets`, and vice
   versa.
3. Every declared secret has a value registered (cross-check
   `connector-builder-secrets list`).
4. Schema validates (Layer B in `connector-configuring`).

Then hand back to `connector-configuring` for the actual `update`
and the next sub-task.

## Common mistakes

| Mistake | Fix |
|---|---|
| Hard-coding a token value into the Configuration "just to test" | Never. Use the secrets store for every secret, including dev. |
| Declaring secrets but not registering them in the store | Always pair declaration with `connector-builder-secrets create`. |
| Guessing OAuth2 grant type when the Tech Overview said ❓ | Go back to Phase 2. Wrong grant type = silent auth failure. |
| Putting credentials in `.env` and assuming the CLI loads them | `.env` is a developer convenience; secrets reach the store only via `connector-builder-secrets create`. |
| Echoing secret values into chat output | Never echo. When troubleshooting, redact. |

## Deeper reference

- `${CLAUDE_PLUGIN_ROOT}/docs/connector-core-knowledge.md §§5–6` — Connections and Secrets concepts.
- `${CLAUDE_PLUGIN_ROOT}/docs/cli-reference.md §5` — `connector-builder-secrets` commands.
- `${CLAUDE_PLUGIN_ROOT}/docs/docs-links.md` — sections "Components & authentication" and "Core configuration concepts."
