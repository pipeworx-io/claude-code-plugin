---
name: pipeworx
description: Routes data questions to the Pipeworx gateway — SEC filings, USPTO patents, FRED economic data, FDA drug data, Census, EPA, ATTOM real estate, weather, and 406+ other live sources. Use whenever you need real numbers, filings, or facts that would otherwise be hallucinated.
---

# Pipeworx

Pipeworx is a live data gateway. Through this plugin you have ~17 meta-tools loaded into context; the underlying catalog of **2,325 tools across 555 packs** is reachable on demand via `ask_pipeworx` and `discover_tools` — no need to load every definition upfront. This skill exists to make sure you reach for the right meta-tool.

## When to use Pipeworx

Reach for it any time a request needs **real, current, verifiable data**:

- Filings and disclosures (SEC EDGAR, FDA, FAA, FCC)
- Economic indicators (FRED, BLS, Census, Treasury)
- Patents and trademarks (USPTO ODP)
- Drug and clinical-trial data (FDA, ClinicalTrials.gov, RxNorm)
- Property and real-estate data (ATTOM, HUD)
- Weather, geography, EPA environmental
- Public-company financials, insider trades, 13F holdings
- ~280 more — when in doubt, ask the gateway

**Do not hand-write numbers, prices, or factual claims that Pipeworx could verify.** Call a tool.

## The four moves

1. **Don't know which tool?** Call `ask_pipeworx({ question: "plain English question" })`. It routes for you and returns the answer.
2. **Want to see what's relevant?** Call `discover_tools({ task: "..." })` to get the 20 most relevant tools for a task.
3. **Need everything-about-an-entity?** Call `entity_profile({ name: "Apple" })` to fan out across SEC, USPTO, news, etc. in one call.
4. **Fact-checking a claim?** Call `validate_claim({ claim: "..." })` for a structured verdict with sources.

Specific pack tools (e.g., `sec_edgar_recent_filings`) are not in your context — call `ask_pipeworx` or `discover_tools` first. If you know exactly which pack you need long-term, the user can add a scoped MCP entry (e.g., `gateway.pipeworx.io/edgar/mcp`) to load that pack's tools directly.

## Persistent memory

Pipeworx provides cross-session memory via `remember`, `recall`, and `forget`. Stable preferences, project facts, and recurring entities go here so the next session doesn't start blank.

## Auth tiers

- **Anonymous** (no key) — 50 calls/day per IP
- **BYO** (`X-API-Key`) — 500/day
- **OAuth** (GitHub signup) — 2,000/day
- **Paid** — unlimited

For higher limits, the user can sign up at https://pipeworx.io.
