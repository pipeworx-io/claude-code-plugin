---
name: prediction-markets
description: Polymarket and Kalshi prediction-market data, plus five Pipeworx-built meta-tools computed on top of it (polymarket_edges, polymarket_arbitrage, polymarket_fill_risk, polymarket_kalshi_spread, polymarket_edge_tracker). Use for "what should I bet on", arbitrage/mispricing scans, cross-venue Polymarket-vs-Kalshi comparisons, and order-book fill risk. The meta-tools are OURS, not the venues' — read this before quoting a number they compute.
---

# Prediction markets: Polymarket, Kalshi, and our own meta-tools

Two raw venues (`polymarket`, `kalshi` packs — market/event/orderbook/trades
lookups) plus five compound tools built on top of both. The compound tools are
where the risk is: they turn raw prices into a probability claim or a trade
recommendation the schema alone doesn't show the mechanics of. This file is
based on reading `handlePolymarketEdges`/`Arbitrage`/`FillRisk`/`KalshiSpread`
in `workers/gateway/src/index.ts`, not the descriptions — though the
descriptions already document most of this well; where true, this file says
so instead of repeating it.

## The calls

| Tool | What it is |
|---|---|
| `polymarket_search` / `polymarket_top_markets` / `polymarket_market` / `polymarket_event` | Raw Polymarket lookups |
| `polymarket_orderbook` / `polymarket_event_books` | Live CLOB order books, single market or whole-event batch (caps at 80 legs) |
| `polymarket_price_history` / `polymarket_trades` / `polymarket_holders` | Time series, fills tape, position holders |
| `kalshi_markets` / `kalshi_events` / `kalshi_event` / `kalshi_series` | Raw Kalshi lookups |
| `kalshi_orderbook` / `kalshi_trades` / `kalshi_price_history` / `kalshi_macro` | Kalshi book, fills, history, macro series |
| `polymarket_edges` | Ranks Polymarket markets where a Pipeworx model disagrees with price |
| `polymarket_arbitrage` | Monotonicity / partition-sum checks within or across Polymarket events |
| `polymarket_fill_risk` | Realizable edge after walking live order-book depth |
| `polymarket_kalshi_spread` | Cross-venue price comparison, Kalshi vs Polymarket |
| `polymarket_edge_tracker` | Day-over-day decay of `polymarket_edges` opportunities |

## What "edge" actually is (verified in `handlePolymarketEdges`)

`edge_pp` is `model_probability − market_probability`, where the model is one
of: a lognormal barrier from 90d FRED log-returns (`crypto_price`), a GDELT
7d-vs-21d news-volume ratio (`news_momentum`, soft signal, half-Kelly), or
partition math on mutually-exclusive legs with a per-sport favorite-longshot
correction (`partition_overround`). There is no fixed horizon — it's model vs
*current* price, and `days_to_resolution` just says how far out the market
resolves. `edge_pp_net` subtracts `slippage_pp` (default 0.3pp, doubled for
partition baskets) once per leg. All in the tool's own description already;
worth adding that `polymarket_edge_tracker` cross-checked cleanly against a
live `polymarket_edges` call today (below) — same market, same `edge_pp_net`.

## Trap: arbitrage and edge numbers are pre-fee, pre-slippage theory — verified from the tool's own output, not just its schema

`polymarket_arbitrage`'s partition check literally says in its `suggested_trade`
text: *"guaranteed 5.0pp edge minus fees/slippage"* (verified live below) — the
number it reports is the raw sum of best-ask/best-bid prices, not what survives
execution. Its own description already says to run `polymarket_fill_risk`
before trading a signal above ~$500, so this isn't undocumented — but it's
easy to miss that "minus fees/slippage" is a caveat, not something already
applied.

**Correction to a common assumption: `polymarket_fill_risk` does model real
depth, not just spread.** It walks the actual CLOB ladder (`walkLadderUsd` /
`walkLadderShares` over every price level) and returns `vwap_fill_price`,
`shares_filled`, and `max_fillable_usd` computed from that walk — not a
top-of-book bid/ask spread. In basket mode it computes `depth_shares` per
leg and flags `thin_legs`/`dead_legs` when a leg can't absorb the requested
size. Confirmed by reading the code (`legDepths`, `walkLadderShares` calls)
and by a live single-market call below.

## Trap: cross-venue spreads match on keyword + structural heuristics, not on resolution source — and that safety check is silently narrower than it looks

`polymarket_kalshi_spread` does not compare the two venues' actual resolution
rules or settlement source anywhere in the code. It picks a candidate pair by
a curated per-topic Polymarket search query plus regex include/exclude
filters, classifies each leg's bet shape (`metric_type`/`match_subtype`) from
its label text, and requires ≥2 shared content tokens to pair a Kalshi leg
with a Polymarket leg. **The tool already discloses this**: in `topic` mode,
a returned pair always carries `compatibility_codes: ["premapped_pairing_unverified"]`
with the sentence *"matched by keyword search and word overlap, not by a
shared resolution source... read each market's rules before sizing anything."*

What is **not** documented anywhere in the schema, and what I verified live:
that warning system — `premapped_pairing_unverified` and `temporal_mismatch`
— only fires in `topic` mode. Calling with explicit `kalshi_event_ticker` +
`polymarket_event_slug` (the mode the description recommends for "custom
pairings") gets `temporal_alignment: null` unconditionally, because the month
comparison is computed only inside the `topic` branch. I paired
`KXFED-26OCT` (Kalshi, resolves Oct 28) against `fed-decision-in-september-762`
(Polymarket, resolves Sep) explicitly and got `"temporal_alignment":null` back
with no mismatch code — verified below. The only safety net left in explicit
mode is `event_subject_mismatch` (zero shared title tokens). If you build a
custom pairing, check the resolution dates and rules yourself; the tool will
not.

## Trap: Kalshi's own trading fees are absent from every comparison

Grepped `mcps/kalshi/src/index.ts` and every `polymarket_kalshi_spread`-adjacent
line in the gateway for "fee" — zero hits. `polymarket_edges`' description
states Polymarket charges no trading fees; nothing anywhere models Kalshi's
real per-contract fee schedule. A spread that looks like a real cross-venue
gap in raw probability terms is comparing a fee-free venue to one that isn't,
with the fee side left at zero on both.

## Verified invocations (live, 2026-09-02, via `scripts/pwcall.sh`)

1. `polymarket_edges({window:"1wk", limit:5})` → `news_momentum` opportunity
   "Iran full airspace closure by September 30?", `model_probability: 0.334`,
   `market_probability: 0.085`, `edge_pp_net: 24.59`.
2. `polymarket_edge_tracker({window:"1wk", days:14})` → same market, same
   `edge_pp_net: 24.59` a day later (`trend: "stable"`) — cross-checks #1.
3. `polymarket_arbitrage({event:"pro-football-2027-champion-20260729185915366"})`
   → `partition_check.arbitrage.overround_pp: 4.95`, 33 legs, `sum: 1.0495`.
4. `polymarket_fill_risk({market:"will-the-los-angeles-rams-win-the-2027-nfl-league-championship", side:"buy_yes", size_usd:2000})`
   → `verdict: "clean"`, `slippage_pp: 0`, `max_fillable_usd: 2425840.69`.
5. `polymarket_kalshi_spread({topic:"fed"})` → resolved `KXFED-26SEP` vs
   `fed-decision-in-september-762`, `temporal_alignment.aligned: true`,
   `matched_pairs: 0` (all 5 Polymarket Fed legs classify as
   `match_subtype:"unknown"` and get excluded — the fed topic essentially
   never produces a scored spread today).
6. `polymarket_kalshi_spread({kalshi_event_ticker:"KXFED-26OCT", polymarket_event_slug:"fed-decision-in-september-762"})`
   → `temporal_alignment: null` despite the Oct/Sep mismatch — confirms the
   explicit-mode gap above.

## A working recipe

1. `polymarket_edges({window})` for candidates; read `by_segment` and each
   opportunity's `edge_pp_net` and `recent_move_warning`.
2. Before trading anything above ~$500, `polymarket_fill_risk({market or
   event, size_usd})` — compare its `realizable`/`vwap` numbers against the
   theoretical `edge_pp`/`overround_pp` you started from.
3. For a specific event's internal consistency, `polymarket_arbitrage({event})`
   for the `partition_check`.
4. For cross-venue, `polymarket_kalshi_spread({topic})` only from the 10
   pre-mapped topics, and always read `compatibility_codes` even when
   `matched_pairs > 0`. For a custom pairing, verify the resolution dates and
   rules text yourself — the tool has no equivalent check outside `topic` mode.

## What not to promise

- No live catalog counts here by design.
- A historical backtest of `polymarket_edges` (59% directional accuracy, 75%
  on BUY signals, 2026-05-22, in fleet memory) was not re-run today — cite it
  as a dated backtest, not a live guarantee; model params may have shifted.
- `polymarket_edges` already excludes Fed/FOMC bets from ranking (unreliable
  FRED-only implied-probability model at meeting-month horizons) — don't
  re-derive a Fed edge from `fed_rate_context` yourself.
