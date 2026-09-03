---
name: filings-and-equities
description: US SEC filings and XBRL fundamentals (EDGAR, sec-xbrl), equity quotes/fundamentals (FMP, Finnhub), retail sentiment (StockTwits), a cross-market recap (market-recap), and A-share market data (china-stocks). Use for "what did $TICKER file", financial statements, insider/institutional holdings, stock quotes and screens, market-wide recaps, or Chinese equity data.
---

# Filings and equities

Six sources, three reliability classes. EDGAR/sec-xbrl: authoritative
primary-source filings, keyless, no rate-limit risk beyond SEC's own. FMP: a
shared platform key on a free tier that upstream keeps shrinking since Aug
2025 — several of its own tool descriptions already say "(paid)"; check the
description before assuming a call works. Finnhub: **bring-your-own-key on
every tool** (`_apiKey` is `required` in the schema) — don't route a caller to
it without one. StockTwits and market-recap: keyless. china-stocks: keyless
with a real coverage gap (below).

## The calls

| Tool | Use it for |
|---|---|
| `edgar_search_filings` | full-text search across all filers |
| `edgar_company_filings` / `edgar_company_facts` / `edgar_company_concept` | one company's filing list / full XBRL dump / one metric over time |
| `edgar_xbrl_frames` | one metric, every filer, one calendar period |
| `edgar_insider_transactions` / `edgar_institutional_holdings` / `edgar_fund_holdings` | Form 4 insider trades / 13F holdings / N-PORT fund holdings |
| `sec_xbrl_get_company_financials` (bare `get_company_financials` also works) | keyless annual-10K snapshot, no key wall ever |
| `fmp_quote` / `fmp_profile` / `income_statement` / `balance_sheet` / `cash_flow` | FMP quote/profile/statements (namespaced names below) |
| `market_snapshot` / `get_quotes` / `price_history` / `top_movers` | market recap, current quotes, historical closes, day's movers |
| `symbol_stream` / `trending_symbols` | StockTwits sentiment feed / trending tickers |
| `ashares_limit_up` / `ashares_quote` / `ashares_analyst_consensus` | China A-share limit-up board / quotes / analyst estimates |

## Trap: the same bare name fails three different ways, and only one is safe

Verified live 2026-09-02, three collisions inside this vertical:

1. `get_company_facts` (sec-xbrl vs `sec` pack) → clean `ambiguous_tool_name`
   error naming `sec_xbrl_get_company_facts` / `sec_get_company_facts`. Safe.
2. `search_filings` (sec-xbrl vs `senate-lobbying`) → same clean error. Safe.
3. `quote` (FMP vs `twelvedata` vs `brapi`) → **plain 200, no error**.
   `quote({symbol:"AAPL"})` and `twelvedata_quote({symbol:"AAPL"})` returned
   byte-identical payloads (`mic_code`, `fifty_two_week` — fields FMP never
   returns); `fmp_quote({symbol:"AAPL"})` returned FMP's own, differently
   shaped data for the same ticker. Silent wrong-pack substitution — same for
   `earnings_calendar` (FMP vs twelvedata), which also resolves silently to FMP.

Nothing predicts which failure mode a bare name gets — **always call the
namespaced form**: `fmp_quote`, `fmp_profile`, `fmp_search_symbol`,
`fmp_earnings_calendar`, `sec_xbrl_get_company_facts`, `sec_xbrl_search_filings`.
`get_company_concept` / `get_company_financials` are uncontested, stay bare.

## Trap: `edgar_search_filings` silently drops a one-sided date filter

`start_date` and `end_date` look independent in the schema; they aren't.
Verified live 2026-09-02, identical query (`"material weakness"`,
`form_type: "10-K"`): `start_date: "2026-08-25"` alone → `total_hits: 10000`
(the unfiltered cap), top hits dated 2020, 2022, 2010 — years before the
requested start. Adding `end_date: "2026-09-02"` → `total_hits: 25`, every
result actually in range. Always pass both.

## Trap (upstream, still live, not reachable through this vertical today)

`efts.sec.gov` — what `edgar_search_filings` calls — caches on `q`/`forms`/
dates but not on `sics`/`ciks`/`entityName`. Verified directly against
efts.sec.gov 2026-09-02: a `sics=6021` (banks) request came back with the
echoed filter still reading `sics: ["7372"]` (software) and
Microsoft/Oracle/C3.ai in the results — the prior request's cache entry,
served under a different filter. No tool here passes `sics`/`ciks`/
`entityName` to efts (`edgar_search_filings` sends only `q`/`forms`/dates), so
this can't be triggered through Pipeworx today — but it's a live defect, and
it will bite the first tool that adds a `sics`/`ciks` filter on top of
full-text search without a cache-busting token in `q`.

## Fixed — say so rather than repeat stale advice

Two SEC XBRL traps that circulated earlier this year (`fy` is the filing year
not the period; `frame` degrades to a company's newest DEF 14A instead of its
last 10-K) do **not** reproduce in `edgar_company_concept` today: it filters
strictly to `ANNUAL_FORMS = {10-K, 10-K/A, 20-F, 20-F/A, 40-F, 40-F/A}` and
picks the row with the latest `end` date, never the raw `fy` field (verified
live below). `edgar_xbrl_frames`, a thinner pass-through of SEC's raw frames
API, never populates `form` at all — always `null` — so it can't tell you
which filing type supplied a cross-company ranking; cross-check a surprising
value with `edgar_company_concept`.

## Trap: a capability that returns nothing, not an error

China A-share capital-flow (主力资金净流入) has no tool here and can't:
`push2.eastmoney.com`'s flow endpoints (`/api/qt/stock/fflow`, `/clist`)
return an empty body from any IP — reverified 2026-09-02, not throttled, just
empty. The `ashares_*` tools use different Eastmoney/Sina endpoints that do
work; capital-flow questions have no answer in this catalog.

## Verified invocations

All run 2026-09-02 through `scripts/pwcall.sh call`, arguments exactly as
shown.

```
edgar_company_concept({ cik: "AMZN", concept: "Revenue", period: "annual" })
```
→ `latest: { fiscal_year: 2025, period_end: "2025-12-31", value: 716924000000,
form: "10-K", filed: "2026-02-06" }`, plus `concept_substituted: true` (Amazon
tags revenue as `RevenueFromContractWithCustomerExcludingAssessedTax`, not
`Revenues`).

```
sec_xbrl_get_company_financials({ company: "NVDA" })
```
→ `entity_name: "NVIDIA CORP", fiscal_year: 2026, revenue.val: 215938000000,
form: "10-K"` — keyless, no FMP/Finnhub key wall.

```
ashares_limit_up({ limit: 5 })
```
→ `limit_up_count: 4`, ranked board with `consecutive_boards`,
`seal_fund_yuan`, `turnover_rate_pct` populated per stock.

## A working recipe: one company, filings + fundamentals + market read

1. `edgar_company_concept({ cik: ticker, concept: "Revenue", period: "annual" })`
   for the headline trend — read `concept_substituted` before calling the
   result "Revenue" verbatim (tickers auto-resolve to CIK).
2. `edgar_insider_transactions({ ticker_or_cik: ticker })` for recent Form 4
   activity — code `P` (open-market buy) is the strongest signal, `A`
   (award) is routine comp.
3. `fmp_quote({ symbol: ticker })` for the current price — always the
   namespaced FMP name, never bare `quote`.
4. `symbol_stream({ symbol: ticker })` for retail sentiment context, with the
   caveat below.

## What not to promise

- StockTwits sentiment tags are self-selected; most messages carry none at
  all, and an untagged message is not a neutral vote — don't compute a
  bullish/bearish ratio over all messages as if silence were neutral.
- FMP's free platform key is shrinking, not stable — an endpoint that works
  today can 402 next quarter. The pack's own error names the workaround
  (bring your own FMP key, or fall back to market-recap/StockTwits) — relay
  it rather than guessing at a fix.
- `edgar_xbrl_frames` ranks are AS-FILED, un-normalized values — a filer's
  unit/scale error shows up as a top-of-list outlier with nothing flagging it.
