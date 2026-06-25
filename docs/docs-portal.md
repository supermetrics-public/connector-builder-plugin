# Supermetrics docs — lookup reference

Companion to `connector-core-knowledge.md` and `cli-reference.md`. Skills
that need specific Configuration JSON structures, examples, or deep-dives
into individual auth flows / pagination / response shapes look them up
**in the live docs**, not in this repo.

The docs are the authoritative source. We deliberately do not vendor
their content here — they evolve, and a copied subset would drift.

## Two sources, used in priority order

| Priority | Source | URL | Audience | Strength | When to use |
|---|---|---|---|---|---|
| 1 (primary) | **Public portal** (Document360) | https://docs.supermetrics.com/docs/connector-builder | Customer-facing | Curated, polished prose; matches the agent's customer-oriented tone | Default for any lookup |
| 2 (fallback) | **Internal Confluence** | https://smdocs.atlassian.net/wiki/spaces/CC/overview | Internal-leaning | Deeper technical detail; structured CQL search returning JSON | When the public portal lacks the technical depth or the topic isn't covered |

Both are publicly accessible without auth. The priority is about
content fit, not access. Use the secondary source when the primary one
falls short, not before.

---

## 1. Primary source — public portal (Document360)

- **Entry point:** https://docs.supermetrics.com/docs/connector-builder
- **Access:** fully public.
- **Scope:** only the **Connector Builder** section of the portal. The
  same site hosts unrelated product docs; ignore those.
- **Platform quirk:** Document360 SPA. The on-page search box renders
  results in JavaScript, which `WebFetch` cannot see — use the recipes
  below, not `?q=…`-style URLs.

### Recipe A1 (primary path) — curated index → `WebFetch`

Before any search, read `docs/docs-links.md`. It's a hand-curated table
of the authoritative Connector Builder articles, grouped by topic.

1. Scan the table for the topic at hand. Pick the most relevant entry.
2. `WebFetch` the URL with a focused prompt:

   ```
   WebFetch("https://docs.supermetrics.com/docs/<slug>", "<focused prompt>")
   ```

3. Article pages **are** server-rendered with real prose — `WebFetch`
   returns useful content. URLs under `/docs/` use **flat slugs**
   (`/docs/authentication`, `/docs/accounts`, `/docs/secrets`, …).

A 404 from `WebFetch` is a signal that the curated index has drifted —
the slug should be repaired in `docs/docs-links.md`.

### Recipe A2 (fallback within primary source) — `WebSearch` with domain filter

When the curated index has no clearly relevant entry but the topic
still feels customer-facing:

```
WebSearch(
  query="<question or topic> connector builder",
  allowed_domains=["docs.supermetrics.com"]
)
```

Always include the literal phrase **"connector builder"** so non-CB
pages on the same domain are deprioritized. When a fallback search
surfaces a useful new article, **add it to `docs/docs-links.md`** so
the next agent finds it via Recipe A1.

If even A2 doesn't surface what's needed, the topic is probably too
technical for the public portal — escalate to source 2.

---

## 2. Secondary source — internal Confluence

- **Entry point:** https://smdocs.atlassian.net/wiki/spaces/CC/overview
  (space key `CC`)
- **Access:** fully public anonymous (no Atlassian login required for
  either the REST API or page URLs).
- **Scope:** the whole `CC` space is Connector Builder; no domain
  filtering needed.
- **Audience tone:** internal/technical. Pages are denser and more
  spec-shaped (e.g. exact object schemas like `MetaInformationObject`,
  adapter configs). The agent should surface relevant pieces in
  customer-friendly language rather than echoing them verbatim.

### Recipe B1 (primary path for this source) — CQL search via REST API

The Confluence REST API is the **real "indexed search"** we wanted —
returns clean JSON, supports the full CQL query language.

```bash
curl -sS \
  'https://smdocs.atlassian.net/wiki/rest/api/content/search?cql=<encoded-cql>&limit=10&expand=title'
```

Typical CQL queries:

| Goal                                           | CQL                                                            |
|-----------------------------------------------|----------------------------------------------------------------|
| All CB pages with "OAuth" in the title         | `space=CC AND type=page AND title~"OAuth"`                     |
| Full-text search for a phrase                  | `space=CC AND text~"refresh token"`                            |
| Restrict to actual articles (skip folders)     | `space=CC AND type=page`                                       |
| Recently updated                               | `space=CC AND type=page ORDER BY lastmodified DESC`            |

Each result includes `id`, `title`, and `_links.webui` — combine with
`https://smdocs.atlassian.net/wiki` for the full URL.

The agent uses `Bash` + `curl` for this; no MCP tools required.

### Recipe B2 — fetch page content

Once CQL returns a candidate page, either of:

- **`WebFetch` on the full page URL.** Confluence pages are
  server-rendered well enough that prose comes back.
- **REST API for clean HTML.** `curl -sS
  'https://smdocs.atlassian.net/wiki/rest/api/content/<id>?expand=body.view'`
  returns the body in `.body.view.value` as already-rendered HTML.
  Useful when the agent wants structured content (tables, code blocks)
  without page chrome.

### Recipe B3 (fallback within secondary source) — `WebSearch`

```
WebSearch(
  query="<question or topic>",
  allowed_domains=["smdocs.atlassian.net"]
)
```

Last resort within this source; CQL is almost always better here.

### URL pattern

Confluence page URLs:

```
https://smdocs.atlassian.net/wiki/spaces/CC/pages/<numeric-id>/<Title+With+Pluses>
```

The CQL response's `webui` field returns a path like
`/spaces/CC/pages/<id>/<Title>` — prepend `https://smdocs.atlassian.net/wiki`
to get the full URL.

---

## 3. Cross-source recipe order

The agent's decision tree when it needs a docs lookup:

1. **A1.** Try the curated index (`docs/docs-links.md`) + `WebFetch`.
   Cheapest and fits customer-tone use cases.
2. **A2.** If no clear match in the index, `WebSearch` the public
   portal with `"connector builder"`. Add a useful new hit back to the
   index.
3. **B1.** If A2 surfaces nothing relevant, CQL-search the Confluence
   space. Pick the top result(s).
4. **B2.** `WebFetch` (or REST `body.view`) the chosen Confluence page.
5. **B3.** Only if CQL itself misses, `WebSearch` `smdocs.atlassian.net`.

Stop as soon as a step produces an answer good enough for the task.

## 4. Tool choice quick map

| Situation                                                                 | Tool / step       |
|---------------------------------------------------------------------------|-------------------|
| Default — any CB docs lookup                                              | A1: `docs-links.md` → `WebFetch`                          |
| Topic isn't in the curated index but is customer-shaped                   | A2: `WebSearch` (public portal, domain-filtered)          |
| Topic is technical / spec-shaped and the portal lacks depth               | B1: CQL via `curl`, then B2                               |
| You have a known Confluence page URL                                      | B2: `WebFetch` (or REST `body.view`)                      |
| You need the most recently changed CB Confluence pages                    | B1 with `ORDER BY lastmodified DESC`                      |
| Public portal's on-page search bar with `?q=…`                            | Do **not** use — SPA, invisible to `WebFetch`             |
| Public portal's `sitemap-en.xml`                                          | Do **not** use — unstructured blob; use `docs-links.md`   |
| Atlassian MCP tools (`mcp__atlassian__*`)                                 | Do **not** use — assumes MCP installed on user's machine; agents should rely on `Bash`/`WebFetch` only |

## 5. What skills should NOT do

- Do **not** scrape Document360 pages with `curl` + HTML parsing. Use
  `WebFetch` — it does markdown conversion cleanly.
- Do **not** reach for the Document360 backend API, hidden search
  endpoints, or auth-token-scraping tricks. (For Confluence, the REST
  API is the documented public path — totally fine to use via `curl`.)
- Do **not** copy doc content into a SKILL.md or any file in this
  repo. The docs change; vendored copies rot.
- Do **not** trust memorized doc contents from the model's training
  data. Always perform a fresh lookup before quoting structures,
  field names, or examples.
- Do **not** echo internal Confluence prose to customers verbatim —
  re-phrase in plain, customer-friendly language. (Source 2 is denser
  and more spec-shaped than the public portal.)

## 6. Failure modes and fallbacks

| Symptom                                                                                  | Likely cause / fix                                                                                       |
|------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| `WebFetch` of a curated slug returns 404                                                 | The slug drifted. Drop to A2 / B1 and repair `docs/docs-links.md`.                                       |
| `WebFetch` of a public-portal page returns mostly empty body                             | Page is SPA-only or redirected. Recheck in a browser; flag to author.                                    |
| `WebSearch` returns mostly non-CB pages                                                  | Add `"connector builder"` (with quotes); add narrowing terms (`"report type"`, `"OAuth2"`).              |
| CQL returns folders or unwanted types                                                    | Add `AND type=page` to the query.                                                                        |
| CQL `webui` URL 404s when WebFetched                                                     | Missing `https://smdocs.atlassian.net/wiki` prefix.                                                      |
| Confluence and public portal contradict each other                                       | The **public portal** is the customer-facing source of truth; defer to it for customer-shaped questions. Use Confluence content only to supplement technical specifics. |
| Confluence prose feels too technical to relay as-is                                      | Distill into customer-friendly language. Internal docs are inputs, not outputs.                          |
| Doc contradicts something in this repo's reference                                       | Docs are authoritative for configuration specifics; this repo is authoritative for the conceptual mental model. Reconcile by updating this repo.                          |
