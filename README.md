# @pipeworx/american-samoa-code

American Samoa Code Annotated (A.S.C.A.) — section lookup by citation and
topic/keyword search. Keyless.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1872+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `as_statute(citation)` — full text of an A.S.C.A. section by citation
  ("46.3502" is Murder in the 1st Degree; "1.0201" is laws applicable in
  American Samoa). Returns the statute text, its chapter, the "History:"
  amendment line, and any "Case Notes:" (case-law annotations) asbar.org
  publishes for that section.
- `as_search(query, limit?)` — topic/keyword search via asbar.org's own
  site search, filtered to Code Annotated section results, returning
  matching citations, headings, and URLs.
- `as_titles()` — all 38 published titles of the Code with their official
  names, for routing a subject to the right title number.

## Auth

Keyless.

## Covering the DC + territories gap

Bruce asked 2026-10-09 to cover DC and the five inhabited US territories
after the 50-state code effort (#2881, #2883-#2885): none of DC, Puerto
Rico, Guam, the US Virgin Islands, American Samoa, or the Northern Mariana
Islands had a code pack (`dc-dmv` is motor-vehicle records only). American
Samoa is the first of the six to ship.

## Source and the one-hop shape

The American Samoa Bar Association (asbar.org) publishes the full Code
Annotated as one WordPress post per section, keyless, no signup — confirmed
live against every tool above. Unlike Oklahoma's OSCN or North Dakota's
ndlegis.gov, there is no per-title index page to resolve a citation against;
instead every section is reachable directly at
`/code-annotated/TITLE-SECTION-slug/`, and the complete citation ->
URL map comes from asbar.org's own Yoast SEO XML sitemaps for the
`code_annotated` post type — confirmed the only route, since that post type
has no REST route (`/wp-json/wp/v2/code_annotated` 404s live). `as_statute`
therefore needs only ONE live fetch per call (the section page itself); the
citation -> URL resolution is a baked lookup, not a second hop.

### Baked index

`src/as-index-data.ts` bakes the citation -> URL map (and the 38 official
title names, from asbar.org's "Code Annotated by Title and Chapter" page)
from the four `code_annotated-sitemap{,2,3,4}.xml` shards — 3,597 URLs,
3,572 citations after resolving a handful of collisions (see below). It
goes through KV, not the gateway bundle (`workers/gateway/src/pack-baked-indexes.json`,
fleet #2754) — the gateway injects it as `args._bakedIndex` on the first
call into this pack.

**Refresh path.** Re-run whenever the Fono amends the Code and asbar.org
adds sections (their schedule, not ours):

```
node mcps/american-samoa-code/scripts/bake-index.mjs > mcps/american-samoa-code/src/as-index-data.ts
```

It fails loudly if either sitemap count or the title count drops well
below what asbar.org serves today — both would mean the site's layout
changed under it.

### Two collision shapes found live (not a parsing bug — asbar.org's own data)

Six of the 3,578 valid section URLs collide on citation. Every collision
was confirmed by fetching both pages' own headings:

1. **WordPress re-publish duplicates** (2 of 6): the identical page
   published twice, the second time at WordPress's own `-2` slug-collision
   suffix (e.g. `6-0210-entitlement-to-vote` and
   `6-0210-entitlement-to-vote-2`, byte-identical statutory text). The
   generator collapses these to the canonical (non-suffixed) URL.
2. **Genuine citation collisions** (4 of 6): two DIFFERENT sections — e.g.
   "4.1402 Definitions." and "4.1402 Individualized service plan–Case
   management." — both published under citation 4.1402. `as_statute`
   returns BOTH as `candidates` with `ambiguous: true` rather than silently
   picking one.

### One irregular slug

`1.0101` ("Citation.") is published at `/code-annotated/10101-citation/` —
no hyphen between title and section, unlike every other URL. Hardcoded as
a single override in the generator.

### Sub-numbered sections

20 of the 3,597 slugs carry a third numeric segment, e.g.
`/code-annotated/4-1307-1-limitation-of-liability/`, which is the literal
citation `4.1307.1` (confirmed live by its own page heading), not part of
the slug's descriptive tail. `as_statute` accepts citations with this third
segment.

### A known upstream content error, left as-is

At least one section's heading and body do not match: the page titled
"4.1307.1 Limitation of liability." (confirmed live) actually serves the
same body text as a different section about departmental agreements. This
is asbar.org's own publication, not a parse error on our side — `as_statute`
returns exactly what the source serves, with the section's own URL, so a
caller can verify.

## Edge probe

Confirmed live from a throwaway `pw-yeva-edgeprobe-americansamoacode`
`wrangler dev --remote` Worker on the prod account (2026-10-09), for all
three URL shapes this pack uses — no Cloudflare Turnstile or other
challenge from a Worker's egress against asbar.org, unlike oscn.net
(`mcps/oklahoma-code`):

| URL shape | Status | Length | First bytes |
|---|---|---|---|
| `/code-annotated/46-3502-murder-in-the-1st-degree/` | 200 | 63,080 B | real HTML doctype + UTF-8 charset meta tag, matching the laptop's own curl |
| `/search/domestic%20violence/feed/rss2/` | 200 | 205,246 B | real RSS 2.0 XML prolog, `channel`/`title` elements present |
| `/code_annotated-sitemap.xml` | 200 | 148,922 B | real sitemap XML prolog, `urlset` element present |

All three `server: cloudflare`, `content-type` matching the real resource
(HTML / RSS+XML / XML) — byte-identical in shape to the laptop's own curl,
confirming asbar.org does not single out Worker egress the way oscn.net
does.

## Data sources

- `https://asbar.org/code_annotated-sitemap{,2,3,4}.xml` — the complete
  list of section URLs (baked once, not fetched per call).
- `https://asbar.org/legal-resources/code-annotated/code-annotated-by-title-and-chapter/`
  — the 38 official title names (baked once).
- `https://asbar.org/code-annotated/TITLE-SECTION-slug/` — one
  section's full text, chapter, history, and case notes. `as_statute`.
- `https://asbar.org/search/<query>/feed/rss2/?paged=N` — the Bar
  Association's own site search, as RSS (paginated, 10 results/page).
  `as_search`.

All keyless, server-rendered HTML/XML — no login, no JS, no rate limit
observed during development.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "american-samoa-code": {
      "url": "https://gateway.pipeworx.io/american-samoa-code/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/american-samoa-code/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1872+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/as_statute \
  -H 'Content-Type: application/json' \
  -d '{"citation":"46.3502"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/as_statute`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "american-samoa-code": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-american-samoa-code"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-american-samoa-code
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about American Samoa Code data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
