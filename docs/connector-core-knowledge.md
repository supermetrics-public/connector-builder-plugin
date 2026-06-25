# Connector Core Knowledge

Shared reference for all Supermetrics Connector-related skills. Defines the
domain vocabulary and the conceptual model a skill author or AI assistant
needs in order to reason about Connectors, Configurations, and the data they
move.

This document is **knowledge**, not instructions. It avoids any "your job
is X" framing so it can be embedded as background context in multiple
skills without conflict.

Source prompts (kept in sync where possible):
- `vasilisa/prompts/10-supermetrics.md`
- `vasilisa/prompts/20-marketing-data.md`
- `vasilisa/prompts/30-connector-builder.md`
- `vasilisa/prompts/40-connector-key-concepts.md`

Formatting convention: canonical terms use **_Term_** (bold + italic) on
introduction and where emphasis aids scanning. This mirrors the existing
Builder assistant prompts so vocabulary stays consistent across surfaces.

---

## 1. Company context

**_Supermetrics_** provides the **_Marketing Intelligence Platform_**, used
by agencies and brands to connect, manage, analyze, and activate their
marketing data. Company site: https://supermetrics.com/.

The platform spans fetching, storage, transformation, analysis, activation,
and delivery to third-party tools. Connector-related skills focus on the
**data pipeline** end of that — specifically, how data enters the platform
from external sources.

## 2. The marketing data pipeline

```
Data Source  →  Connector  →  Marketing Intelligence Platform  →  Destination
   (API)        (config)         (storage / analysis)           (sheet, BI, DWH)
```

### 2.1 Data Sources (inputs)

A **_Data Source_** is any external location or service from which marketing
data can be fetched.

- Typically marketing, e-commerce, or advertising platforms providing data
  on campaign efficiency, engagement, reach, and other KPIs.
- Range from huge general platforms (Google Ads, Facebook) to highly
  specialized niche services.
- Vendor dashboards exist, but **reliable automated access usually goes
  through the vendor's API**. Less convenient fallbacks (email, shared
  files) exist for some sources.
- Every Data Source has its own bespoke API. Without a platform like
  Supermetrics, each customer would have to build or buy custom integration
  software per source.

### 2.2 Destinations (outputs)

A **_Destination_** is the final point in the pipeline: a component that
delivers data — previously fetched and processed by the platform — to a
third-party service.

Examples shipped today:
- Supermetrics for Looker Studio
- Supermetrics Google Sheets add-on
- Supermetrics for Microsoft Power BI
- Supermetrics for Excel

### 2.3 Storage and warehouses

When a Data Source is slow or rate-limited, retrieved data can be persisted
inside the platform for efficient downstream analysis. The dedicated storage
layers are called **_DWH_** and **_Supermetrics Storage_**.

## 3. Connectors

### 3.1 What a Connector is

A **_Connector_** is a specialized software component that provides
automated access to one distinct Data Source.

- Acts as a **data transformation layer**: maps the raw response of the
  Data Source's API (typically JSON or CSV) into the platform's internal
  structure.
- **Stateless**: does not permanently store data between requests beyond
  caching used for performance.
- The platform ships ~hundreds of pre-built Connectors. Thousands more
  sources are routinely requested — too many for the vendor to build all
  itself. **_Connector Builder_** exists so customers can build their own.

### 3.2 Connector Configuration

Connectors are not hand-coded per source. Supermetrics maintains an
underlying platform that generalizes API fetching: pluggable
authentication, request building, response parsing, error handling, etc.

Building a new Connector is therefore **composing the right
_Connector Configuration_**, not writing code. This is what classifies the
tool as low-code: structured composition, no programming, but enough
inherent complexity (because data and source APIs are complex) that
guidance and good defaults matter.

### 3.3 Report Types — the unit of output

The output of a Connector is always a selection of **_Report Types_**.
Each Report Type is a single, integrated blueprint that specifies:

- How to build the **_Request_** to the Data Source API.
- How to parse the **_Response_** that comes back.
- The resulting data structure of **_Fields_** — the Dimensions and
  Metrics that the Report Type exposes (see §4).

A Connector typically defines multiple Report Types so users can pick the
shape of data that fits their analysis.

## 4. Marketing data: Reports, Metrics, Dimensions

Marketing data in Supermetrics products is represented as a **report** (or
collection of reports): the structured, retrievable unit that defines a
dataset.

Every report decomposes into two complementary kinds of field:

- **_Metrics_** (the values) — quantifiable numbers being measured.
  Examples: Clicks, Impressions, Cost, Conversions, Revenue. These carry
  the core numerical weight.
- **_Dimensions_** (the context) — attributes that categorize and segment
  metrics, narrowing what each number means. Examples: Date, Campaign
  Name, Country, Ad ID, Placement Type.

A Metric without a Dimension is a single context-free number. The report
is the tabular output where Metrics are made meaningful by the Dimensions
selected alongside them.

### 4.0a What is NOT a Report

Reports are about **measuring**, not about retrieving a specific
record. A Report Type produces tabular data with meaningful
Dimensions and (typically) Metrics. The following are **not**
Report Types and must not be modeled as such:

- **Single-entity lookups** — `GET /users/123`, `GET /campaigns/456`.
  These return one record; they have no aggregation and no
  meaningful Dimensions beyond the record's own ID. A Connector is
  not the right tool for "fetch a specific record."
- **Mutating endpoints** — POST/PUT/PATCH/DELETE. Connectors are
  read-only on the data plane.
- **Authentication / session endpoints** — `/login`, `/me`,
  `/token/introspect`. These belong inside auth or Connection
  identity, not Reporting.
- **List endpoints that exist solely to support other requests** —
  e.g. account-listing endpoints belong in the Accounts section, not
  Reporting.

A useful rule: _"Could a marketer plot this as a chart or pivot it
into a table?"_ If no, it's probably not a Report Type. If yes, it
is.

A degenerate but valid case is a Report Type with one row (e.g.
"current account balance"). What makes it a Report is that it has
meaningful Metrics, not that the result set is large.

### 4.1 Example reports

- **Paid Advertising Performance:** Metrics like Cost, Conversions, ROAS
  broken down by Dimensions like Date, Campaign Name, Device Type.
- **Social Media Content Performance:** Metrics like Reach, Engagement
  Rate, Shares broken down by Dimensions like Date, Media Type, Post URL.

## 5. Authentication: Connections and Accounts

### 5.1 Connections

Most Data Source APIs require authentication: Requests must carry security
tokens or other secrets, typically as parameters or headers.

The full context required for an authorized session with the source is a
**_Connection_**. A Connection holds:

- Security details: access tokens, refresh / exchange tokens, expiration
  periods.
- Other contextual data: external session IDs, scope tags, etc.

Before a Connector can be used, a user initiates a Connection by either
providing credentials directly or completing an auth flow (OAuth1, OAuth2,
or custom). **The specifics of that flow — and the exact data persisted in
the resulting Connection — are entirely determined by the Connector
Configuration.**

### 5.2 Accounts

An **_Account_** is an independent scope of marketing data accessible
under a single Connection. Example: multiple websites under one analytics
profile, or multiple ad accounts under one ads login.

A Connector can fetch data for one, many, or all available Accounts. The
Configuration defines:

- How to **list** the Accounts available under a Connection.
- Which **attributes** of each Account can be passed into subsequent
  Requests.

---

## 6. Secrets management

Connectors almost always need **secret values** — API keys, OAuth client
secrets, signing keys, long-lived refresh tokens, etc. **Secrets are
never hard-coded into the Connector Configuration.** The system uses
ID-based references in the config, with values held in a dedicated store
managed by the CLI tool.

### 6.1 Two-sided registration

A secret used by a Connector must be registered in **two** places:

1. **In the Configuration** — a root-level **`secrets`** object declares
   every secret ID the Connector references.
2. **In the CLI secrets store** — every declared ID must have a value
   registered through the CLI before the Connector can run.

If an ID exists on one side but not the other, the Connector won't
function. Authors should keep both sides in sync as they iterate on
Phase 4.

### 6.2 CLI surface (conceptual)

The CLI tool provides commands for:

- **Registering** a new secret (`id → value`).
- **Replacing** the value of an existing secret.
- **Listing** registered secrets (typically IDs and metadata only — never
  the values themselves).

This document captures the **workflow**. For the exact subcommands
(`connector-builder-secrets create / update / list / delete`), masked-input
behavior, and required flags, see `docs/cli-reference.md §5`.

### 6.3 Local-development convention

During development, secret values typically live in a local **`.env`**
file inside the per-connector project folder.

⚠️ **`.env` is a skill-imposed convention, not a CLI feature.** The CLI
does not auto-load values from a project `.env`. The user (or a small
helper script) still has to register each secret into the store via
`connector-builder-secrets create` — the `.env` just keeps the values
ready to hand and out of the configuration file. See
`docs/cli-reference.md §5` for the rationale and the registration
command.

The per-connector project folder layout therefore extends to:

```
<connector-name>/
  technical-overview.md
  samples/
  config.json
  logs/
  .env                  # local secret values — NEVER committed
  .gitignore            # at minimum: .env
```

### 6.4 Hygiene rules

Skills working with Connectors should actively enforce:

- **Never commit `.env`** or any file containing real secret values to
  version control. The skill should remind the user to add `.env` to
  `.gitignore` (and create `.gitignore` if it doesn't exist).
- **Never inline secret values into the Configuration**, even
  temporarily for testing. Always go through the secrets store.
- **Never echo secret values** into chat output, commit messages, or log
  files. When dumping requests/responses during troubleshooting, redact
  known secret fields.
- **Prefer "rotate" over "reuse"** when a value may have been exposed —
  use the CLI's replace command rather than reasoning about whether
  exposure was harmful.

---

## 7. Recommended workflow for building a Connector

This is the proven shape of the work, distilled from experience inside
Supermetrics. Connector-related skills are organized around these phases,
and AI assistants should encourage users (and themselves) to follow them
in order. Skipping ahead — especially writing configuration before the
data and endpoints are understood — is the most common cause of wasted
effort and broken connectors.

### Phase 1 — Define the scope

A Data Source is rarely a single API. Big platforms expose entire
**families of APIs**, each with its own contract, fields, and quirks. A
goal like _"build a connector to Facebook"_ is therefore ill-posed: it
doesn't say which API or which data.

Good scope statements name the platform **and** the kind of data:

- "Fetch typical campaign efficiency data from platform N."
- "Scrape a rarely changing JSON dictionary from data source M."
- "Pull product-level sales data from platform X."

The scope drives every later decision (which docs to read, which auth, which
Report Types to define), so it must be explicit and agreed before moving on.

#### Two entry paths into Phase 1

There are two common starting situations, and the skill should recognize
which one applies:

**Greenfield — "I want to fetch X from Y."**
The user describes what data they want from which platform. Scope has to
be elicited through conversation: which data exactly, at what
granularity, for what purpose. This is the path the prose above assumes.

**Brownfield — "I already do this, but manually / by script."**
The user already has a working solution and wants it productized as a
Connector. Possible inputs include anything the user has handy:

- A vibe-coded integration script (Python, Node, etc.) — typically an
  AI-generated one-file program that already pulls the data.
- A `curl` command or a shell snippet they run by hand.
- A Postman / Insomnia / Bruno collection.
- A small dashboard or reporting project that already calls the API.

This is **easier**, not harder: a working artifact is a high-quality
specification. The skill should read the user's code/script/HTTP calls
and **extract** the scope from it — endpoints used, auth method, fields
referenced, pagination handled — rather than asking the user to
re-articulate it from scratch.

> **The vibe-coded integration case is the headline brownfield path
> and the plugin's competitive differentiator vs the web UI.** The
> typical input is _"Claude wrote me a Python script that pulls X from
> Y and it works; turn it into a real Supermetrics Connector."_ The
> scoping skill prioritizes recognizing this shape and frames the
> conversion as _"you've proven the integration works; let's make it
> production-grade."_ Other input shapes are supported but treated as
> secondary.

Brownfield work also pre-pays parts of later phases:

- The endpoints, fields, and auth method feed directly into the **Phase 2**
  Technical Overview.
- Real responses the script has been receiving can be saved as
  **Phase 3** verification samples without making new requests.
- If the script works, the agent can be confident the Data Source
  exposes what's needed and that the auth/parsing approach is viable.

The handoff back to the standard workflow happens after extraction:
the agent presents the scope it inferred (and the endpoints, fields,
auth it identified) for the user to confirm — then proceeds with
Phases 2–5 as usual, just with most of Phase 2 already drafted.

### Phase 2 — Research and write the Technical Overview

With the scope fixed, study the Data Source's public documentation and
answer two questions:

1. **Does the source actually expose the data we want?** Vendor products
   often show data in dashboards that the public API does not return.
2. **Can our platform consume it?** The Supermetrics platform supports a
   defined set of authentication mechanisms, request signing methods, and
   response shapes. Some APIs publish data in formats that the
   configuration's parser can't handle. This is rare, but it happens, and
   it's cheaper to discover here than after writing configuration.

The output of this phase is a **_Technical Overview_** document. Minimum
contents:

- **Connector name** and a one-line **purpose** statement (the scope).
- Short description of the **Data Source platform**.
- The **endpoints** to consume, with **links to the upstream docs**.
- The **fields** each endpoint returns, mapped to the data we need.
- The **authentication** method(s) the API requires.
- **Errors**: what the API returns on common failure cases.
- **Rate limits / quotas**, if any.
- **Ideally:** full sample requests with sample responses.

The Technical Overview is the contract between research and
configuration — and it is the **single highest-leverage artifact** in
the workflow. A good one makes Phase 4 a mostly mechanical translation
task; a bad one guarantees rework, and no amount of cleverness during
configuration will recover from missing or wrong information here.

**Implications for skill design.** The Phase 2/3 skill should be at
least as detailed and rigorous as the Phase 4 skill, not less. Time
the agent spends ensuring the Tech Overview is complete, accurate, and
user-confirmed pays back many times during Phase 4.

### Phase 3 — Verify with real requests

Before configuring anything, hit the chosen endpoints directly (curl, an
HTTP client, the API's own sandbox) and confirm:

- Authentication works against a real account.
- The endpoints return the fields the Tech Overview claims.
- The data shown matches what the user expects to see.

Where appropriate, show the raw response to the user for **visual
confirmation** — "yes, this is the data I wanted."

Save the verified sample responses **alongside the Technical Overview**.
These become reference data for Phase 4 and Phase 5.

#### Per-connector project folder

Supermetrics does not host any of these artifacts. AI-assisted skills
should advise creating a dedicated **per-connector project folder**
under the user's current working directory. Layout at minimum:

```
<connector-name>/
  technical-overview.md     # the Tech Overview from Phase 2
  samples/                  # raw request/response captures from Phase 3
  config.json | config.yaml # the configuration written in Phase 4
                            # (JSON today, YAML once the CLI accepts it)
  logs/                     # optional: validation-run artifacts from Phase 5
  .env                      # local secret values — NEVER committed (§6)
  .team-id                  # cached --team-id for this project
  .ds-id                    # the saved Connector's identifier (5-char) — written after first create
  .ds-user                  # the authenticated Login id for the Data Source — written in Phase 5 setup
  .gitignore                # at minimum: .env, .team-id, .ds-id, .ds-user
```

The folder is created under whatever directory the Claude Code
session started in — no fixed root. The folder is the working
context for all subsequent phases.

The skill asks for `--team-id` on the first CLI call that needs it
(or detects it via `connector-builder list` if only one team is
visible), stores the answer to `.team-id`, and reads from that file
silently for the rest of the session.

### Phase 4 — Write the configuration

A Connector Configuration is a structured document — currently a JSON
object that can run to **thousands of lines** for a real connector.
A JSON Schema exists; in the future the format is expected to move to
**YAML** with a simplified structure, so skills should focus on **concepts
and structure**, not on JSON-specific syntax details.

At a high level, a configuration is made up of several **conceptual**
sections. These are the mental model — the JSON Schema may name or
group them differently and is expected to evolve, but the model
endures.

- **Authentication and Connection** — how to authorize, what Connection
  data to persist.
- **Secrets** — declarations of every secret ID the Connector uses
  (values live in the CLI's secrets store, never in the config; see §6).
- **Accounts** — how to list and select Accounts under a Connection.
- **Reporting** — the Report Types: Requests, Response parsing, Fields.
- **Response handling** — shared parsing / pagination / error handling.
- **Optional sections** for optional features.

> **Note on schema terminology.** The current JSON Schema also exposes
> optional root objects such as `info` and `metaInformation`. These are
> not part of the customer-facing model and are not surfaced by the
> skills unless explicitly asked for. The term **"Hybrid Connector,"**
> which appears in the schema title, is outdated and is not used by any
> skill — the product is **Connector Builder**, the artifact is a
> **Connector**.

Phase 4 is best decomposed by **configuration task**, not by schema
section. A schema may regroup or rename its root objects between
versions; the tasks themselves are stable. The typical task set:

- Configure **Auth + Connection**.
- Configure **Accounts** (how to list, how to select).
- Configure **one Report Type** — performed once per Report Type the
  Connector exposes; a Connector typically has several.
- Configure **default response handling** (shared parsing, pagination,
  error handling).
- Configure any **optional / source-specific features** (rate-limit
  handling, custom field transforms, etc.).

Skills built around Phase 4 should mirror this task split. They should
**not** be organized around the JSON Schema's root property names, and
should not surface schema-only artifacts (e.g. `info`, `metaInformation`)
to the user unless explicitly asked.

Practical realities of this phase:

- **Authentication is the hardest part for most users**, especially OAuth2
  and any flow involving token exchange or refresh.
- Saving the configuration via the CLI requires correct **syntax and
  structure**, so authors should expect to **iterate several times** —
  fix → save → fix → save — before the file is accepted.
- The configuration **must live in a file**; the CLI takes that file as a
  parameter (`--configuration-file`) for `connector-builder update`.
  Today the file is `config.json`; the format is moving to YAML
  (`config.yaml`) in the near term. The skill detects format from the
  file extension and reads/writes accordingly. The much smaller
  connector-metadata wrapper (`name`, `description`) is passed inline
  as a string, not kept as a separate file.
- **Every save is effectively a publish.** There is no draft / published
  separation today; an `update` either succeeds (and the new
  Configuration is live) or errors (and nothing changes). When `create`
  is first called, the platform seeds the new Connector with a valid
  boilerplate Configuration so the slot is never invalid. The
  configuring skill leans on its pre-flight checklist plus the schema
  validator to keep every `update` clean; see `docs/cli-reference.md §13`.

If the Technical Overview is solid, this phase is a **translation
task** — turning a well-specified contract into structured config. The
configuring skill is, in essence, a translator: it should rely on the
Tech Overview as its source of truth rather than re-deriving
information. If the Overview is missing details, the right move is to
go back to Phase 2/3 and fix it — not to guess in config.

### Phase 5 — Validate

A saved Connector can be **executed via the same CLI tool** that saved
it — but only after a **Login** (Connection) exists for its Data
Source. A Login is created via the CLI's `login-links create` flow:
the CLI returns a URL, the user opens it in a browser, completes the
Data Source's authentication, and a Login record appears that
`queries execute` can use. Without a Login, even a perfect
Configuration fails with auth errors.

Once a Login exists, validation has two layers:

1. **Doesn't blow up.** The Connector should run end-to-end without
   random errors.
2. **Returns the right data.** Run the Connector and compare its output
   against the raw responses captured in Phase 3. Mismatches mean
   parsing, field mapping, or pagination is wrong.

When something is off, **troubleshooting** is the dominant activity:

- The CLI exposes **request/response logs** for executed Connectors. These
  are the primary diagnostic source — far more informative than the
  configuration alone.
- Common failure modes include auth-token refresh, pagination cursors,
  response shape drift, and field-mapping mistakes.

Troubleshooting is also valuable outside this workflow — a user with an
existing broken or misbehaving Connector lands here directly.

---

## Glossary (quick lookup)

| Term | Meaning |
|---|---|
| **Supermetrics** | The company; provider of the Marketing Intelligence Platform. |
| **Marketing Intelligence Platform** | Supermetrics' end-to-end platform for marketing data: connect → store → analyze → activate → deliver. |
| **Data Source** | External service/platform from which marketing data is fetched (usually via API). |
| **Connector** | Software component that provides automated, transformed access to one Data Source. Stateless. |
| **Connector Configuration** | The composed spec that, loaded by the platform, becomes a Connector. The artifact a customer authors in Connector Builder. |
| **Connector Builder** | The low-code tool / web app for authoring Connector Configurations. |
| **Report Type** | A single retrievable blueprint inside a Connector: Request + Response parsing + Fields. |
| **Report** | The tabular output users consume: Metrics × Dimensions. |
| **Metric** | Quantifiable value being measured (Clicks, Cost, Conversions). |
| **Dimension** | Categorizing attribute that gives a Metric context (Date, Campaign, Country). |
| **Field** | A Metric or Dimension exposed by a Report Type. |
| **Request** | The outbound call a Connector makes to a Data Source API. |
| **Response** | The raw payload the Data Source returns; the Connector parses it. |
| **Connection** | Full authorization + contextual state for using a Connector against a source. |
| **Account** | An independent scope of data under one Connection (e.g. one ad account of many). |
| **Destination** | Component that delivers processed data to a third-party tool (Looker Studio, Sheets, Power BI, Excel, etc.). |
| **DWH / Supermetrics Storage** | Internal storage layers used when Data Sources are slow or rate-limited. |
| **Technical Overview** | Pre-config document describing the connector's purpose, target endpoints, fields, auth, errors, and rate limits, with sample requests/responses. The contract between research and configuration. |
| **Project folder** | Per-connector working directory created under the user's CWD. Holds the Technical Overview, samples, config file, validation artifacts, `.env`, and `.team-id`. Not hosted by Supermetrics. |
| **Secret** | A sensitive value (API key, OAuth client secret, signing key, refresh token) referenced by ID inside the configuration. The value lives in the CLI's secrets store, never in the config file. |
| **Secrets store** | The CLI-managed store where secret values are kept. Accessed via `register` / `replace` / `list` commands. |
| **`.env`** | Convention for holding local secret values inside the project folder during development. Must not be committed. |
