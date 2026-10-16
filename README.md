# @pipeworx/cn-law

Mainland China's national laws, administrative regulations, local
regulations, judicial interpretations (司法解释), and the Supreme People's
Court's official Guiding Cases (指导性案例) — original Chinese text, with
promulgation and effective dates kept as separate fields where the source
provides them.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1815+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `cn_search_laws(query, exact?, category?, limit?)` — search national laws,
  administrative regulations, local regulations, and judicial
  interpretations by title (exact) or keyword (fuzzy). Returns each result's
  record id, title, category, issuing body, promulgation date, effective
  date, and in-force status.
- `cn_law_detail(id)` — metadata for one record from `cn_search_laws`:
  title, category, issuing body, promulgation date, effective date, in-force
  status, and whether `cn_law_article_text` can extract text for it.
- `cn_law_article_text(id, article)` — the Chinese text of ONE specific
  article/provision (by number or citation, e.g. `13` or `"第十三条"`).
  Returns `found: false` with an explanation — never a fabricated or
  truncated snippet — when the article cannot be located or the underlying
  document cannot be read as text.
- `cn_search_admin_regulations(query, limit?)` — search administrative
  regulations by title, from a second official source whose detail pages
  carry the full regulation text inline (rather than the PDF route
  `cn_law_article_text` uses).
- `cn_admin_regulation_text(law_id)` — full Chinese text of one
  administrative regulation found via `cn_search_admin_regulations`, plus
  its revision-date history.
- `cn_guiding_cases_list(page?)` — index of the Supreme People's Court's
  official Guiding Cases, newest batch first.
- `cn_guiding_case_detail(id)` — full text of one Guiding Case: keywords
  (关键词), holding (裁判要点), and the full case narrative.

## Auth

Keyless. No login, no API key, for all three sources.

## Data sources

- NPC Legislative Affairs Commission (全国人大常委会法工委, National
  People's Congress Standing Committee), National Database of Laws and
  Regulations — national laws, administrative regulations, local
  regulations, and judicial interpretations (SPC/SPP/jointly issued), with
  promulgation and effective dates as separate fields. The publisher's own
  site is a JavaScript single-page app with no documented API; this pack
  calls the JSON endpoints that app itself calls, found by reading its
  bundle. Article-level text is not a field this source exposes directly —
  only a signed document-viewer link and highlighted search snippets — so
  `cn_law_article_text` extracts it from the same official PDF rendition the
  site itself serves, via the shared `extractPdfText` helper
  (`@pipeworx/shared`).
- Ministry of Justice (司法部), National Administrative Regulations
  Database — a second official copy of administrative regulations, whose
  detail pages carry the full text inline as plain HTML. Its title search
  silently ignores the query term unless a second, undocumented parameter
  is also sent — `cn_search_admin_regulations` asserts every returned title
  actually contains the query before returning it, so that failure mode
  cannot come back silently.
- Supreme People's Court (最高人民法院), official Guiding Cases (指导性案例)
  column — a static, paginated index with one full-text page per case.

**Not used, by design:** the newer People's Court Case Library
(人民法院案例库) and China Judgments Online (中国裁判文书网). Both sit
behind a login wall for the calls that matter (the case library's search API
answers "not logged in" with no session); this pack does not walk around
that.

## Known gaps

- `cn_law_article_text` extracts article boundaries by finding the next
  `第N条` heading after the requested one (a real heading is followed by a
  full-width space in these PDFs; a cross-reference to another article
  mid-sentence is not). For the LAST article in a document, or an unusually
  long span, the response's `note` field says so — read it before trusting a
  long or last-article result.
- `cn_admin_regulation_text` carries a revision-date history but not
  separate promulgation/effective-date FIELDS the way `cn_search_laws` does
  — those are only stated inside the regulation's own text (e.g. "(2001年
  12月25日...公布...)"), consistent with how this source actually publishes
  the document.
- Neither source offers an English translation; there is nothing to label
  as official/unofficial because none exists.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "cn-law": {
      "url": "https://gateway.pipeworx.io/cn-law/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/cn-law/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1815+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/cn_search_laws \
  -H 'Content-Type: application/json' \
  -d '{"query":"个人信息保护法","exact":true}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/cn_search_laws`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "cn-law": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-cn-law"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-cn-law
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Cn Law data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
