# Pipeworx Claude Code Plugin

Connect Claude Code to live data from **2,325 tools across 555 packs** — SEC filings, USPTO patents, FRED economic data, FDA drug data, Census, EPA, ATTOM real estate, weather, and 547+ more.

Backed by the [Pipeworx](https://pipeworx.io) MCP gateway at `gateway.pipeworx.io`.

## Install

```text
/plugin marketplace add pipeworx-io/claude-code-plugin
/plugin install pipeworx
```

## How it works

The plugin loads **17 meta-tools** into Claude's context — not all 2,325 underlying tools. That's deliberate: dumping every tool definition into the context window burns tokens you'll never use (the "context tax").

Instead, Claude reaches for `ask_pipeworx` or `discover_tools` and the gateway routes the request to the right pack at session time. You get the full catalog without paying for it up front.

The loaded meta-tools:

- **`ask_pipeworx`** — natural-language router. *"What's Apple's latest 10-K?"* hits SEC EDGAR. *"Side effects of Ozempic?"* hits FDA. No knowing pack names needed.
- **`discover_tools`** — top-20 most relevant tools for a task, with full schemas. Use when you want to see options before committing.
- **`entity_profile`**, **`recent_changes`**, **`compare_entities`**, **`resolve_entity`** — fan-out across multiple packs in one call.
- **`validate_claim`** — fact-check claims against SEC XBRL. Returns a verdict + citation.
- **`remember`** / **`recall`** / **`forget`** — persistent memory across sessions.
- **`list_packs`**, **`search_packs`**, **`get_pack_tools`**, **`get_connection_config`**, **`get_platform_status`**, **`search_mcp_directory`** — browse the catalog.

The bundled skill teaches Claude when to reach for each.

## Verify after install

In Claude Code, run:

```text
/mcp
```

You should see `pipeworx` connected with ~17 tools. Then try:

> What was the unemployment rate last month?

Claude should call `ask_pipeworx` (which routes to `fred_get_series`) and return a real number.

## Need direct pack access?

If you want a specific pack's tools loaded directly (e.g., to call `attom_property_search` without going through `ask_pipeworx`), add a scoped MCP entry:

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

Or a vertical bundle (e.g., `?vertical=housing` for the housing data stack).

## Higher rate limits

The plugin runs on the anonymous tier (50 calls/day per IP). For higher limits (500/day BYO, 2,000/day OAuth, or unlimited paid), [sign up at pipeworx.io](https://pipeworx.io) and configure your own MCP entry with an `X-API-Key` header.

## Links

- Gateway: https://gateway.pipeworx.io
- Stack guide: https://pipeworx.io/stack
- Source: https://github.com/pipeworx-io/pipeworx

## License

MIT
