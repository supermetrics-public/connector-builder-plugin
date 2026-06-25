# Curated Connector Builder docs index

Hand-curated index of the authoritative Supermetrics docs articles for
Connector Builder. This is the **first place** any Connector-related
skill looks when it needs specifics on Configuration JSON, auth flows,
report types, error handling, etc.

How the agent uses this file:

1. Scan the table for the topic at hand and pick the most relevant
   article(s).
2. `WebFetch` the URL(s) and reason over the returned prose. Article
   pages are server-rendered, so `WebFetch` returns real content.
3. Only if nothing here clearly matches, fall back to the search
   recipe in `docs/docs-portal.md §2`.

Maintenance:

- Curated by hand. Add entries when a useful article is discovered;
  remove entries when a slug 404s.
- **Slug drift is the main rot risk** — a `WebFetch` returning 404 is
  the signal to fix the URL here.
- Descriptions should be short and topic-anchored, written so the
  agent can pick on title + description alone.

---

## 1. Introduction & getting started

| Article | URL | What it covers |
|---|---|---|
| Connector Builder (landing) | https://docs.supermetrics.com/docs/connector-builder | Section landing page. Index of CB topics. |
| Getting started | https://docs.supermetrics.com/docs/getting-started-with-connector-builder | High-level walkthrough of the build flow. |
| Basic concepts | https://docs.supermetrics.com/docs/basic-concepts | Vocabulary and mental model (companion to `connector-core-knowledge.md`). |
| Accessing Connector Builder | https://docs.supermetrics.com/docs/accessing-connector-builder | Who can use CB, where it lives in the product. |
| Using custom connectors | https://docs.supermetrics.com/docs/using-custom-connectors-in-supermetrics | Consuming a finished custom connector in Supermetrics products. |
| First-connector tutorial | https://docs.supermetrics.com/docs/tutorial-for-your-first-connector | End-to-end walkthrough; useful as a reference example. |
| Supported APIs & technologies | https://docs.supermetrics.com/docs/supported-apis-and-technologies | What kinds of source APIs CB can talk to (auth methods, request shapes, response formats). Read in Phase 2 to confirm feasibility. |

## 2. Components & authentication

| Article | URL | What it covers |
|---|---|---|
| Components | https://docs.supermetrics.com/docs/components | The configurable building blocks of a Configuration. |
| Placeholders | https://docs.supermetrics.com/docs/placeholders | Placeholder/templating syntax used across the Configuration. |
| Authentication (overview) | https://docs.supermetrics.com/docs/authentication | Start here for any auth question — overview of supported methods. |
| API key authentication | https://docs.supermetrics.com/docs/api-key-authentication | Configuring sources that authenticate via a static API key. |
| OAuth2 authentication | https://docs.supermetrics.com/docs/oauth2-authentication | Configuring OAuth2 flows (the most common and hardest auth case). |
| OAuth1 authentication | https://docs.supermetrics.com/docs/oauth1-authentication | Configuring legacy OAuth1 flows. |

## 3. Core configuration concepts

| Article | URL | What it covers |
|---|---|---|
| Request signers | https://docs.supermetrics.com/docs/request-signers | Signing outbound requests (HMAC, etc.). |
| Data source user | https://docs.supermetrics.com/docs/data-source-user | The user / identity context a Connection represents. |
| Accounts | https://docs.supermetrics.com/docs/accounts | Listing and selecting Accounts under a Connection. |
| Reporting | https://docs.supermetrics.com/docs/reporting | Defining Report Types — request, response parsing, fields. |
| Error handling | https://docs.supermetrics.com/docs/error-handling | Mapping HTTP / platform errors to Connector behavior. |
| Request configuration | https://docs.supermetrics.com/docs/request-configuration | Building outbound requests (path, params, headers, body, pagination). |
| Secrets | https://docs.supermetrics.com/docs/secrets | Declaring and referencing secrets inside the Configuration (paired with the CLI secrets store). |

## 4. Advanced configurations

| Article | URL | What it covers |
|---|---|---|
| Advanced configurations | https://docs.supermetrics.com/docs/advanced-configurations | Overview of less-common configuration features. |
| Joining data | https://docs.supermetrics.com/docs/joining-data | Combining results from multiple requests. |
| Transforming response structure | https://docs.supermetrics.com/docs/transforming-response-structure | Reshaping a response before fields are mapped. |

## 5. Examples

| Article | URL | What it covers |
|---|---|---|
| Example configurations | https://docs.supermetrics.com/docs/example-configurations | Reference Configurations the agent can study before authoring its own. |

---

## Open follow-ups

- **Descriptions are currently inferred from slugs**, not from the
  article bodies. Replace each "What it covers" cell with a one-line
  summary taken from the actual article body when convenient. (Each
  is a single `WebFetch`; can be done in one batch.)
- **Coverage gaps will surface during skill authoring.** When a
  configuring sub-skill needs a topic that isn't in this table, add it
  here rather than letting the agent rediscover it via search.
