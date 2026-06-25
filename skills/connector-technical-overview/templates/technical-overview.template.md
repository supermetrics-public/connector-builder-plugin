# {{Connector name}}: Technical Overview

> Phase 2 artifact for the Supermetrics Connector build workflow.
> Captures what the Connector needs to do, what API surface it talks
> to, and what evidence we have that the API actually supports it.
> Once this document is complete and the user has confirmed it
> (Phase 3 — verification), it becomes the contract that drives
> Phase 4 (configuration).
>
> **How to use this template.** Fill in the bracketed placeholders.
> Delete sections that genuinely don't apply (e.g. the Date / timezone
> section for sources without notable date handling). Keep the section
> order so reviewers know where to look.

## Header

- **Connector name:** {{name}}
- **Purpose:** {{one-line scope statement from Phase 1}}
- **Data Source platform:** {{platform name + landing URL}}
- **Date of overview:** {{YYYY-MM-DD}}

## Section conventions

Each capability section uses a **status emoji** in the heading:

- ✅ — Confirmed working, no concerns
- ⚠️ — Works but has caveats or risks
- ❓ — Needs further investigation
- ❌ — Not feasible or blocked

Each section follows this pattern:

1. **API documentation link(s)** — inline at the top of the section.
2. **How it works** — brief description of the mechanism.
3. **Implementation notes** — facts the configuring phase will need.
4. **Example request** — curl or equivalent.
5. **Example response** — sanitized JSON / XML (no real secrets).
6. **Concerns** — risks, limitations, open questions.

Write findings, not investigation logs. Be concise — a reader should
be able to scan the entire overview in under 5 minutes and understand
the verdict.

---

## Authentication {{status emoji}}

API docs: {{URL}}

**Method:** {{API key in header / API key in query parameter / Basic auth / OAuth2 (specify grant type) / OAuth1 / JWT / Cookie / Custom / Multiple}}

**How it works:**
{{Short mechanism description.}}

**Implementation notes:**

- Token / credential endpoint URL and request body format (JSON vs form-encoded):
- Required permission scopes:
- Token field name in the response and header name used for authenticated requests:
- Token expiry duration and whether re-authentication is needed:
- IP allowlisting required / optional / n/a:
- User setup steps (creating an app, enabling permissions, …):

**Example request:**

```bash
{{curl example}}
```

**Example response:**

```json
{{sanitized JSON}}
```

**Concerns:**
{{… or "none"}}

---

## Connection identity (Data Source User) {{status emoji}}

API docs: {{URL or "n/a"}}

Defines how the authenticated token and user-specific information are
stored alongside the Connection.

- Is there a `/me`, `/token/introspect`, or equivalent endpoint? Which one?
- Fields that need to be stored alongside the token (user ID, email,
  account / portal ID, instance URLs, …):
- How is the access token referenced in subsequent data requests?

If no `/me`-style endpoint exists, note that user input will be
required to capture the identity.

---

## Accounts {{status emoji}}

API docs: {{URL}}

Defines how multiple Accounts under one Connection are listed and
selected.

- Account-listing endpoint URL:
- Is account selection hierarchical (e.g. agency → client → property)?
- Fields per account (ID, name, currency, …):
- Pagination on the listing endpoint:
- **Single-account APIs:** if the source has no Account concept, note
  "connection-as-account" and explain.

**Example account-listing request:**

```bash
{{curl}}
```

**Example account-listing response (sanitized):**

```json
{{JSON}}
```

---

## Reporting

One sub-section per **Report Type** the Connector will expose. Each
sub-section MUST include an example request and a sanitized example
response — these are the primary evidence that the API was tested
against live data.

### Report Type: {{report name}} {{status emoji}}

API docs: {{URL}}

**Endpoint and request:**

- HTTP method, URL, required parameters:
- Which parameters map to Dimensions vs Metrics:
- Date filtering parameters and format:

**Example request:**

```bash
{{curl}}
```

**Response structure:**

- JSONPath / XPath to the data rows:
- Field names and types:
- Fields requiring transformation (timestamps, padding, currency
  conversion, …):

**Example response (sanitized):**

```json
{{JSON}}
```

**Field coverage:**

| Field | Status | Notes |
|---|---|---|
| {{field}} | ✅ Available | direct from API response |
| {{field}} | 🔢 Calculated | derived from {{other fields}} |
| {{field}} | ⚠️ Needs join | requires {{secondary request}} |
| {{field}} | ❌ Not available | not exposed by API |

**Concerns:**
{{e.g. data completeness risks, response size limits, dynamic field
schemas — or "none"}}

_(Repeat the above sub-section once per Report Type.)_

If the API supports **segments** (user-defined templates, lists, custom
objects), add a sub-section describing the segment-selection endpoint
and how segment IDs flow into the Report Type's data requests.

---

## Pagination {{status emoji}}

API docs: {{URL}}

| Pattern                         | Used by this API | Details                                      |
|---------------------------------|------------------|----------------------------------------------|
| Offset / limit                  | {{yes/no}}       | param names; max page size                   |
| Page number                     | {{yes/no}}       | 0-indexed or 1-indexed                       |
| Cursor / next-page-token        | {{yes/no}}       | where the cursor lives in the response       |
| `Link` header (RFC 5988)        | {{yes/no}}       |                                              |
| No pagination                   | {{yes/no}}       | note max response size limits                |
| Custom / session-based          | {{yes/no}}       | flag — may need investigation                |

**Hard result caps** (absolute upper bound on total rows regardless of
pagination): {{… or "none"}}

---

## Rate limits and quotas {{status emoji}}

API docs: {{URL}}

- Limits per second / minute / hour / day:
- Whether limits are per API key, per account, or per endpoint:
- Rate-limit headers returned (e.g. `X-RateLimit-Remaining`):
- Behavior on limit hit: 429 with `Retry-After`, silent throttle,
  error code, …:
- Data freshness delays (e.g. reporting data available T+3 hours):

Per-Report-Type variances: note in the relevant Report Type sub-section
above.

---

## Date range and timezone handling {{status emoji}}

Include only if the source has notable date-handling behavior.

- Server-side date-range filtering supported:
- Parameter format (ISO 8601, Unix timestamp, …):
- Exclusive vs inclusive end date:
- Timezone parameter support and format:
- Default timezone if no parameter is available:

---

## Errors {{status emoji}}

API docs: {{URL}}

- Common error shape (HTTP code + body structure):
- Common error scenarios users will hit (auth expiry, invalid date
  range, no data, account access lost, …):
- Whether errors are returned with 2xx status (some APIs do this —
  flag explicitly):
- Retryable vs terminal classification:

---

## Summary

**Verdict — one of:**

- **Feasible** — every required capability is supported by Connector
  Builder.
- **Feasible with caveats** — supported, with one or more compromises
  (list them below).
- **Not feasible** — list the specific blockers below.

**What works:**

- {{bulleted list of confirmed capabilities}}

**Caveats / limitations:**

- {{bulleted list, or "none"}}

**Known risks:**

| Risk | Severity | Impact | Mitigation |
|---|---|---|---|
| {{description}} | low / med / high | {{...}} | {{...}} |

**Field coverage summary:**

| Report Type | Available | Calculated | Needs join | Not available |
|---|---|---|---|---|
| {{name}} | n | n | n | n |
