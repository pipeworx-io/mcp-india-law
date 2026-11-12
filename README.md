# @pipeworx/india-law

Indian statutes (central and state acts, one section's text at a time),
full-text search over about 1.1 million provisions of legislation and regulator
instruments, and full-text search over the text of Supreme Court of India and
High Court judgments.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1879+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `india_search_acts(...)` — find acts by title words. Central acts by
  default; set `jurisdiction` to a state name for state acts. Returns the
  `act_id`, year, act number, ministry, enactment and in-force dates, and a
  repealed flag.
- `india_get_section(...)` — fetch the text of one section: pass the act title
  (or `act_id`) and the section number, e.g. "Indian Contract Act, 1872" s.73.
  **IPC requests:** the Indian Penal Code, 1860 was repealed from 2024-07-01
  and replaced by the Bharatiya Nyaya Sanhita, 2023 (BNS), and the repository
  no longer carries IPC sections individually. Asking for "Section 302 IPC"
  returns the BNS successor section (s.103, "Punishment for murder") with its
  full text, marked `text_is_successor: true`, for a curated list of
  well-known offences (murder, culpable homicide, rape, theft, cheating,
  cruelty to a wife s.498A, defamation, and similar). The list is a subset,
  not the full concordance, and a BNS section may be a sub-section of the
  number given; an IPC section outside the list returns `found:false` with a
  pointer to `india_search_sections`. The Code of Criminal Procedure and the
  Indian Evidence Act are likewise replaced (BNSS and BSA, 2023) and are
  reported as repealed-and-replaced.
- `india_search_sections(...)` — search section headings and text across acts
  (e.g. "murder"), optionally limited to one act by `act_id`.
- `india_search_provisions(...)` — full-text search over the TEXT of 1,098,269
  provisions: 544,078 from central and state legislation (10,112 acts) and
  554,191 from the instruments of fifteen regulators (12,152 instruments: RBI,
  SEBI, MCA, IRDAI, TRAI, CBIC, DGFT and others). Finds what the law says
  about a subject ("anticipatory bail", "limitation period for suits"), ranked
  by BM25. Optional `jurisdiction` (`central`, a state or union territory such as
  `tamil-nadu`, `states`, a regulator such as `rbi`, or `regulations`) or
  `act_id` scope the search. Every result is one provision with its act,
  section, chapter, an excerpt around the matched words, and the publisher's
  link. Match is all-words first, falling back to any-word for a sentence.
- `india_get_provision(...)` — the full text of one section by act title and
  section number ("Information Technology Act, 2000", "43A"), with its
  sub-sections in order; `"8F(2)"` selects one sub-section. A missing section
  returns the sections that act does have. The dataset
  does not carry every section of every act (it holds 118 sections of the
  Indian Contract Act, 1872), so `india_get_section`, which reads the
  repository live, is the fallback.
- `india_search_judgments(...)` — full-text search over the TEXT of Indian
  court judgments: the Supreme Court of India and 25 High Courts,
  32,572,660 passages of judgment text (about 15.8 million judgments; a judgment is stored as overlapping
  passages of about 4,000 characters), 1950 onward. Ranked by BM25; all words
  first, falling back to any word. Optional `court` (`supreme-court`, or a
  High Court by name or slug such as `delhi`, `bombay`, `punjab-and-haryana`)
  and `year` or `from_year`/`to_year`. One result per judgment, at its
  best-matching passage: case title, court, decision date, citation (where the
  court records one), case number, CNR, disposal, an excerpt around the matched
  words, and a `ref`. The year filter with no court asks every court.
- `india_get_judgment(...)` — the stored text of one judgment by `ref`, with
  the passage overlaps removed; long judgments are returned in pieces of
  `max_chars` with `next_start_chunk` to continue. The text is the publisher's
  extraction from the court's published judgment: it can contain extraction
  noise (repeated fragments, line-break loss) and the court's own record is
  authoritative.
## Judgments index

`india_search_judgments` queries a full-text index of 32,572,660 passages (SQLite
FTS5 in ten shards, about 36.0 GB) and reads text from R2 (25.7 GB
compressed, 64 passages to a gzip block, found by arithmetic on the hit's
id). A court, and a court's year range, is one id range, so filters are
native to the index. Stand-alone installs of this pack have neither, and the
two tools answer `search_unavailable`. The load is
`scripts/india-law/ingest_judgments.py` (plan, run, verify, manifest); the
data is a dated snapshot (`data_as_of` in every response).

## Auth

Keyless.

## Data sources

- Open India Law (Vaquill AI), legislation and regulations dataset, CC BY 4.0
  (snapshot 2026-08). Central and state acts come from India Code; the
  regulator instruments from each regulator's publications. Attribution is
  carried in every response: *Source: Open India Law (Vaquill AI), CC BY 4.0*,
  with `data_as_of`. Text is the publisher's snapshot and may not reflect later
  amendments; it is not legal advice.

- Open India Law (Vaquill AI), court judgments dataset, CC BY 4.0 (snapshot
  2026-08): the Supreme Court of India and 25 High Courts. Every judgment
  response carries *Judgments: Supreme Court of India and the High Courts of
  India, via Open India Law (Vaquill AI), CC BY 4.0* and `data_as_of`; credit
  the originating court when reusing.

- Legislative Department, Government of India, India Code — central and state
  acts and their sections. Queried live per request.
Statute text is as published by the Legislative Department and may lag
amendments and is not legal advice. Dates: acts carry `enacted` and
`in_force_from`.

## Index

The two provision tools query a full-text index (SQLite FTS5, four shards,
about 0.46 GB) and read provision text from R2 (0.26 GB compressed, one bundle
per act). A standalone install of this pack has neither, so those two tools
answer `search_unavailable` and point at the live India Code tools. The load is
`scripts/india-law/build_legislation.py` then `scripts/ingest-india-law.mjs`.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "india-law": {
      "url": "https://gateway.pipeworx.io/india-law/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/india-law/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1879+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/india_search_acts \
  -H 'Content-Type: application/json' \
  -d '{"query":"contract"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/india_search_acts`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-india-law` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/india-law/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "india-law": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-india-law"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about India Law data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
