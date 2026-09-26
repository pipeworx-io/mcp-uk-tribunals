# UK Tribunal Decisions — GOV.UK

First-instance and Upper Tribunal decisions published on GOV.UK: the
**first-tier Employment Tribunal**, the Employment Appeal Tribunal, the
Residential Property Tribunal, the Upper Tribunal (Administrative Appeals
Chamber — social security, disability and child support appeals), and the
Upper Tribunal (Tax and Chancery Chamber).

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1684+ live data sources.

**The reason this pack exists:** Find Case Law (`uk-caselaw`) carries the
Employment **Appeal** Tribunal only — it has no court code for the first-tier
Employment Tribunal. So a search there for "employment tribunal decisions"
silently returns EAT appeals, and the 134,000+ first-tier ET decisions GOV.UK
actually publishes were unreachable anywhere in the catalog before this pack.
`uk-caselaw` remains the right pack for appeals and every other UK court.

## Auth

**None.** Keyless GOV.UK search + content API.

## Tools

| Tool | Use it when |
|---|---|
| `search_tribunal_decisions` | You have a subject, party name or date range and know which tribunal |
| `get_tribunal_decision` | You have a `link` from a search result and want the full text |
| `list_tribunal_categories` | You want the exact category slug to filter by (not guessable from the label) |

The usual path is `search_tribunal_decisions` → take a result's `link` →
`get_tribunal_decision`.

## The five tribunals

| `tribunal` arg | What it is | Decisions (2026-09-18) |
|---|---|---|
| `employment` | Employment Tribunal, first-tier | 134,136 |
| `employment_appeal` | Employment Appeal Tribunal | 2,615 |
| `residential_property` | Residential Property Tribunal / FTT (Property Chamber) | 18,229 |
| `administrative_appeals` | Upper Tribunal (Administrative Appeals Chamber) | 2,079 |
| `tax` | Upper Tribunal (Tax and Chancery Chamber) | 1,442 |

GOV.UK also publishes a generic `decision` content type (~13,000 items) —
**deliberately not exposed here**. It is teacher-misconduct panel outcomes,
medicine licence suspensions and other professional-regulator decisions, not
tribunal decisions; routing a tribunal question there would silently answer it
with the wrong kind of document.

## Data shape traps worth knowing

- **Full text is not published for every decision.** `get_tribunal_decision`
  returns `text_available: false` when GOV.UK has not extracted text — this is
  the norm for Tax and Chancery Chamber decisions, which are PDF-only. `pdf_url`
  is always returned when a PDF exists.
- **The category field name is not the same across tribunals.** Employment,
  Employment Appeal and Administrative Appeals use a plural array
  (`tribunal_decision_categories`); Residential Property and Tax use a singular
  string (`tribunal_decision_category`). Filtering `search_tribunal_decisions`
  by `category` against the wrong field name doesn't error upstream — it just
  matches nothing, which reads as "no such category" rather than "wrong field".
  This pack routes on the right field per tribunal internally; `category` takes
  the same slug regardless of which field backs it.
- **`list_tribunal_categories` reads live**, off each tribunal finder page's own
  `details.facets` in the GOV.UK content API — not a hardcoded list. GOV.UK adds
  and renames categories over time; a hardcoded list would go stale silently.

## Data sources

- Search: `https://www.gov.uk/api/search.json?filter_content_store_document_type=<type>`
- Item: `https://www.gov.uk/api/content<link>` — `details.metadata.hidden_indexable_content`
  is the full extracted decision text where published; `details.attachments[]`
  carries the original PDF(s) on `assets.publishing.service.gov.uk`.
- Category facets: `https://www.gov.uk/api/content/<finder-base-path>` —
  `details.facets`.

Crown copyright, published under the
[Open Government Licence v3.0](https://www.nationalarchives.gov.uk/doc/open-government-licence/version/3/).

## Caveats worth passing to a user

- **England, Wales and Scotland only, as published on GOV.UK.** Northern Ireland
  tribunals are not on this platform.
- **Decisions before roughly 2017 are sparse.** GOV.UK publication of tribunal
  decisions is not retrospectively complete — an older decision's absence means
  *not published here*, not *no such decision*.
- **This is not a court.** No Crown Court, County Court or Magistrates' Court
  decisions are published here — those are first-instance courts, not
  tribunals, and their decisions (mostly oral) are not published anywhere in
  this form.
- **Search is over the decision text itself**, so a phrase the tribunal would
  actually write (a party name, a case number) finds more than a legal concept
  it never names.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "uk-tribunals": {
      "url": "https://gateway.pipeworx.io/uk-tribunals/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/uk-tribunals/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1684+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/search_tribunal_decisions \
  -H 'Content-Type: application/json' \
  -d '{"tribunal":"employment","query":"disability discrimination","limit":3}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/search_tribunal_decisions`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "uk-tribunals": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-uk-tribunals"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-uk-tribunals
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Uk Tribunals data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
