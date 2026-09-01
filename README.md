# Pipeworx for Claude Code

Give Claude one MCP that reaches **5,635+ live-data tools across 1,477+ sources** — SEC filings, USPTO patents, FRED, Census, FDA, EPA, USAspending, Polymarket, Zillow, weather, and 1,469+ more — without loading 5,635+ tool schemas into your context window.

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

The plugin exposes **~31 meta-tools**, not all 5,635+ — `ask_pipeworx({question})` and friends route at runtime so you get the full catalog without paying the context tax for tools you'll never call this session.

## Free tier + signup

**Signing in is free and takes one GitHub click** — it moves you from 50 calls a day to 200, on a stable account that does not rotate with your IP. Point the server at `https://gateway.pipeworx.io/oauth/mcp` and complete the sign-in when prompted, or [sign up first](https://pipeworx.io/signup?via=cc_plugin).

No account at all still works: `https://gateway.pipeworx.io/pipeworx-catalog/mcp`, anonymous, 50 calls a day per IP.

## Verify after install

```text
/mcp
```

You should see `pipeworx` connected with ~38 tools.

## What's loaded

- **`ask_pipeworx`** — natural-language router across all 1,477+ sources.
- **`discover_tools`** — top-20 relevant tools for a task, with full schemas.
- **`entity_profile`** / **`compare_entities`** / **`recent_changes`** / **`resolve_entity`** — fan-out across multiple packs in one call.
- **`validate_claim`** — fact-check claims against SEC XBRL.
- **`remember`** / **`recall`** / **`forget`** — persistent memory across sessions.
- **`list_packs`** / **`search_packs`** / **`get_pack_tools`** / **`get_connection_config`** / **`get_platform_status`** / **`search_mcp_directory`** — browse the catalog.

The bundled skills teach Claude when to reach for each.

## Bundled skills

Skills load on demand — Claude reads one when the work matches its description, so they cost nothing until they are relevant.

| Skill | What it covers |
|---|---|
| `pipeworx` | Which meta-tool to reach for, and when |
| `gtm-funding-signals` | Target lists from SEC Form D — recent raises, repeat founders, amendment de-duplication |
| `gtm-hiring-signals` | Companies hiring, from Adzuna postings and H-1B LCA filings |
| `gtm-company-resolution` | Messy company names → CIK / LEI / FIGI / Companies House number, and domain verification |
| `gtm-email-hygiene` | Keyless MX / SPF / domain-age scrub before you spend a verification credit |
| `gtm-derived-signals` | Composing the above into a scored account list — the signal before the signal |
| `gtm-headless` | Calling tools with no MCP server: the `pipeworx` CLI and plain HTTP JSON-RPC |

The six `gtm-*` skills are written for GTM engineers working from a terminal. Every call in them was run against the live gateway and the response pasted in, including the arguments that are silently ignored — so you inherit the traps instead of rediscovering them.

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
