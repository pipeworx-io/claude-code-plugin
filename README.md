# Pipeworx for Claude Code

Give Claude one MCP that reaches **3,300+ live-data tools across 750+ sources** — SEC filings, USPTO patents, FRED, Census, FDA, EPA, USAspending, Polymarket, Zillow, weather, and 740+ more — without loading 3,300+ tool schemas into your context window.

## Install

```text
/plugin marketplace add pipeworx-io/claude-code-plugin
/plugin install pipeworx
```

## Try it

After install, ask Claude things like:

| Ask | What it triggers |
|---|---|
| *"What just happened to Apple?"* | `sec_8k_recent` → SEC 8-K events classified by severity |
| *"Spread between Polymarket and Kalshi on the next Fed decision?"* | `polymarket_kalshi_spread` → live cross-venue mispricing |
| *"Overdue Phase 3 readouts at Moderna?"* | `pharma_pipeline_catalysts` → biotech catalyst calendar |
| *"DoD cybersecurity contracts this week?"* | `usa_award_search` → sub-second USAspending mirror |
| *"Median home value and renter share in Lubbock, TX?"* | `housing_market_snapshot` + `housing_metro_demand` |
| *"Unemployment rate last month?"* | `fred_get_series` → official FRED data |

Claude picks the right tool via `ask_pipeworx` — no pack-name memorization required.

## How it loads light

The plugin exposes **~26 meta-tools**, not all 3,300+ — `ask_pipeworx({question})` and friends route at runtime so you get the full catalog without paying the context tax for tools you'll never call this session.

## Free tier + signup

100 calls/day anonymous, IP-bound (rotates on networks). [Sign up free in 10s via GitHub](https://pipeworx.io/signup?via=cc_plugin) for 2,000/day + a stable account that doesn't rotate.

## Verify after install

```text
/mcp
```

You should see `pipeworx` connected with ~26 tools.

## What's loaded

- **`ask_pipeworx`** — natural-language router across all 750+ sources.
- **`discover_tools`** — top-20 relevant tools for a task, with full schemas.
- **`entity_profile`** / **`compare_entities`** / **`recent_changes`** / **`resolve_entity`** — fan-out across multiple packs in one call.
- **`validate_claim`** — fact-check claims against SEC XBRL.
- **`remember`** / **`recall`** / **`forget`** — persistent memory across sessions.
- **`list_packs`** / **`search_packs`** / **`get_pack_tools`** / **`get_connection_config`** / **`get_platform_status`** / **`search_mcp_directory`** — browse the catalog.

The bundled skill teaches Claude when to reach for each.

## Direct pack access

For a specific pack's tools loaded directly (e.g., `attom_property_search` without going through `ask_pipeworx`), add a scoped MCP entry:

```json
{
  "mcpServers": {
    "pipeworx-attom": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://gateway.pipeworx.io/attom/mcp"]
    }
  }
}
```

Or a vertical bundle (e.g., `?vertical=housing` for the housing-data stack).

## Links

- Gateway: https://gateway.pipeworx.io
- Status: https://pipeworx.io/status
- Source: https://github.com/pipeworx-io/pipeworx

## License

MIT

---

⭐ Star if you'd use this — helps other Claude Code users discover it.
