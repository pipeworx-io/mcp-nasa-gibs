# @pipeworx/nasa-gibs

NASA GIBS — the ~1,300-layer global satellite imagery catalogue behind NASA
Worldview, with daily true-colour mosaics, thermal anomalies, sea-surface
temperature, aerosols, snow and ice cover and night lights, most of them with a
multi-decade daily time series.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1679+ live data sources.

## Tools

- `gibs_list_layers(query?, projection?, limit?)` — search the catalogue by subject or
  identifier; returns each layer's id, title, grid and available date range.
- `gibs_layer_info(layer, projection?)` — one layer's full time coverage, native
  resolution, format, extent, colour map and legend URLs, and tile template.
- `gibs_tile_url(layer, date?, latitude?, longitude?, zoom?, tile_row?, tile_col?, projection?)` —
  a ready-to-fetch image tile URL, with the tile's own geographic bounds.

## Auth

Keyless. No registration, no rate-limit headers.

## Data sources

- <https://gibs.earthdata.nasa.gov/wmts/{projection}/best/1.0.0/WMTSCapabilities.xml> —
  the layer catalogue, one document per projection (`epsg4326`, `epsg3857`,
  `epsg3413`, `epsg3031`).

### Things the next person would otherwise rediscover

- **There is no JSON layer index.** The WMTS capabilities XML is the only
  machine-readable catalogue and it is ~5 MB per projection. It is parsed once
  per isolate and the (much smaller) result cached in module scope for 6 hours;
  a per-call re-parse would spend seconds of CPU on a document that changes
  about once a day. The cache is populated only inside a request — never at
  module load, where the Workers runtime reports `Date.now()` as 1970 and every
  entry would read as infinitely stale.
- **Each layer has several `<ResourceURL resourceType="tile">` templates.** Only
  one carries `{Time}`; that is the one worth handing a caller. The others drop
  the date silently and serve whatever GIBS considers current.
- **`MatrixWidth` is not a power of two** (levels go 2, 3, 5, …), so tile
  row/column cannot be derived from the zoom level alone. They are computed from
  `ScaleDenominator` the way WMTS specifies: `tileSpan = ScaleDenominator ×
  0.00028 m × TileWidth ÷ metresPerUnit`, with `metresPerUnit` = 111319.49 for
  `epsg4326` (degrees) and 1 for the metre-based projections.
- **`TopLeftCorner` is written x-then-y** (`-180 90`), not the lat-then-lon axis
  order EPSG:4326 formally prescribes.
- **Lat/lon → tile conversion is offered for `epsg4326` and `epsg3857` only.**
  The two polar stereographic grids need `tile_row`/`tile_col` directly, and the
  tool says so rather than returning a wrong tile.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "nasa-gibs": {
      "url": "https://gateway.pipeworx.io/nasa-gibs/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/nasa-gibs/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1679+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/gibs_list_layers \
  -H 'Content-Type: application/json' \
  -d '{"query":"fires","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/gibs_list_layers`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "nasa-gibs": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-nasa-gibs"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-nasa-gibs
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Nasa Gibs data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
