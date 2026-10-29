# polymarket-us

Market data from **Polymarket US** (polymarket.us) — the CFTC-regulated,
US-licensed prediction-market exchange. This is a SEPARATE venue from the
offshore Polymarket platform (see the `polymarket` pack) and from Kalshi (see
the `kalshi` pack): different company, different markets, different slugs. A
slug from one venue never resolves on another, and every response here
carries `venue: "polymarket-us"` plus `data_as_of` so a caller comparing
across packs never silently mixes venues.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

Upstream: `https://gateway.polymarket.us` (docs:
<https://docs.polymarket.us/api-reference/introduction>). This is Polymarket
US's **public** market-data host — no auth, no account, no API key for any
endpoint this pack calls. A separate authenticated host, `api.polymarket.us`,
handles order placement and portfolio/account data; this pack is read-only
market data only, same posture as `polymarket` and `kalshi`.

## Tools

| Tool | What it answers |
|---|---|
| `polymarket_us_search` | find markets by keyword (`query`), or browse the highest-volume open markets by category |
| `polymarket_us_bbo` | top-of-book: current price, best bid/ask, depth levels, open interest |
| `polymarket_us_book` | full bid/ask depth ladder plus session stats (open/high/low/last-trade price) |
| `polymarket_us_price_history` | Yes/No display-price time series, by fixed interval + fidelity |
| `polymarket_us_settlement` | the final settlement price once a market has resolved; `settled: false` while open, `error: "not_found"` (never "still open") for a slug that doesn't exist |

All five need a market `slug`, which `polymarket_us_search` returns.

## Auth

Keyless. The gateway.polymarket.us market-data endpoints need no API key,
account or authentication of any kind — verified live, including from a
Cloudflare Workers edge location (`wrangler dev --remote`), since the
sibling `kalshi` pack discovered its upstream filters Cloudflare egress on
some endpoints and this one does not share that problem.

## Data sources

- `GET /v1/search` — full-text search across events/markets.
- `GET /v1/markets` — filterable market listing (used for category/volume
  browsing when `polymarket_us_search` is called without a `query`).
- `GET /v1/markets/{slug}/bbo` — best bid/offer (lightweight top-of-book).
- `GET /v1/markets/{slug}/book` — full order book + session stats.
- `GET /v1/price-history?symbol={slug}&fidelity=…&fixedInterval=…` —
  historical Yes/No display prices.
- `GET /v1/markets/{slug}/settlement` — settlement price; 404s until the
  market resolves. A bare 404 is ambiguous (open market vs. nonexistent
  slug), so this pack disambiguates with a cheap `GET /v1/markets?slug=`
  existence check before reporting `settled: false` for a market that
  exists, or `error: "not_found"` for a slug that doesn't — never "still
  open" for a market that was never real.

## Notes

- Price history's `longPrice`/`shortPrice` (mapped here to `yes_price` /
  `no_price`) are **display** prices — `yes_price` is normally derived from
  the best ask and `no_price` from one minus the best bid — so they preserve
  the bid/ask spread and CAN sum to more than 1.0. That is expected, not a
  data error.
- The settlement endpoint returns a price only, no timestamp. **Resolution-
  audit integration (fleet #1909) is explicitly out of scope** for this pack
  — it would need (a) a stable per-market settlement timestamp, which this
  API does not expose, and (b) a bulk "settled since &lt;date&gt;" filter, which
  `/v1/markets` also does not have (only a boolean `closed` flag) — without
  one, auditing settlements means polling every known slug.
- `/v1/search` has no category filter (only `seriesIds`, `marketType`,
  time-range and `status`). `polymarket_us_search`'s `category` argument is
  therefore browse-mode only (routes to `/v1/markets`) and is silently
  ignored when `query` is set — the tool description says so.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "polymarket-us": {
      "url": "https://gateway.pipeworx.io/polymarket-us/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/polymarket-us/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1856+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/polymarket_us_search \
  -H 'Content-Type: application/json' \
  -d '{"query":"fed funds","limit":3}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/polymarket_us_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-polymarket-us` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/polymarket-us/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "polymarket-us": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-polymarket-us"]
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
ask_pipeworx({ question: "your question about Polymarket Us data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
