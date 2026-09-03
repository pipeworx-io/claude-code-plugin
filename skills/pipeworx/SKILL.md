---
name: pipeworx
description: Routes data questions to the Pipeworx gateway — SEC filings, USPTO patents, FRED economic data, FDA drug data, Census, EPA, ATTOM real estate, weather, and 1,469+ other live sources. Use whenever you need real numbers, filings, or facts that would otherwise be hallucinated.
---

# Pipeworx

Pipeworx is a live data gateway. Through this plugin you have ~31 meta-tools loaded into context; the underlying catalog of **5,635+ tools across 1,477+ packs** is reachable on demand via `ask_pipeworx` and `discover_tools` — no need to load every definition upfront. This skill exists to make sure you reach for the right meta-tool.

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

## Two tools can want the same name, and it resolves three different ways

A generic tool name — `quote`, `search_dockets`, `search_notices`,
`get_company_facts` — is often claimed by more than one pack. What happens when
you call the bare name is **not consistent**, and one of the three outcomes is
silent. Verified 2026-09-02:

| Bare name | What happens |
|---|---|
| `search_dockets`, `search_notices` | `Unknown tool` — call the namespaced form (`court_listener_search_dockets`, `ted_eu_search_notices`) |
| `get_company_facts`, `search_filings` | a clean `ambiguous_tool_name` error naming the candidates |
| `quote` | **a 200 — one pack silently wins.** `quote({symbol:"AAPL"})` returns twelvedata's payload; `fmp_quote` returns a different vendor's, in a different shape |

That third row is the dangerous one. You get real, well-formed data from a
source you did not choose — and for market data, vendors differ in timing,
coverage and licensing.

**So never call a generic bare name and assume you know the source.** Resolve it
first with `discover_tools({ task: "..." })`, which returns names as actually
exposed, and prefer the prefixed form (`fmp_quote`, `twelvedata_quote`) whenever
one exists. An `Unknown tool` on a plausible name is a naming artefact, never a
statement about coverage.

Specific pack tools (e.g., `sec_edgar_recent_filings`) are not in your context — call `ask_pipeworx` or `discover_tools` first. If you know exactly which pack you need long-term, the user can add a scoped MCP entry (e.g., `gateway.pipeworx.io/edgar/mcp`) to load that pack's tools directly.

## Persistent memory

Pipeworx provides cross-session memory via `remember`, `recall`, and `forget`. Stable preferences, project facts, and recurring entities go here so the next session doesn't start blank.

## Auth tiers

- **OAuth** (sign in with GitHub — free, one click) — 200/day
- **BYO** (`X-API-Key`) — 200/day
- **Anonymous** (no key, no account) — 50 calls/day per IP
- **Paid** — unlimited

For higher limits, the user can sign up at https://pipeworx.io.
