---
name: auctions-and-surplus
description: US auction and government-surplus listings with realized sale prices — search ~1M live lots across ~1,900 auction houses plus GovDeals, AllSurplus, GSA and PublicSurplus, find what closes soon near a ZIP, and price an asset off what things actually sold for. Use for "what is this equipment worth", surplus vehicle and machinery hunting, or any resale/valuation question.
---

# Auctions and surplus

This is a hosted, deduplicated store, not a passthrough to one site. Two things
follow: the numbers are ours to explain, and the comps are the reason to be here
— **realized hammer prices**, which most listing sites never expose.

Call `us_auctions_coverage({})` first when a question turns on scale or
freshness. It returns per-source lot counts and `last_refreshed_at` per source,
so you can say what you are standing on instead of implying totality.

Live 2026-09-02: **987,204 distinct active lots** out of 1,044,070 raw,
**56,866 cross-listed**, 2,611 open auction events, 537,511 retained sold comps.

## Pick the right surface — they overlap

| Tool family | Scope | Use it when |
|---|---|---|
| `us_auctions_*` | **everything, deduplicated** | the default; always start here |
| `auctions_*` | government sources only | you specifically want public-agency surplus |
| `gsa_*` | GSA federal sales only | federal-specific questions |

The `us_auctions_*` family is the one that de-duplicates. **GovDeals and
AllSurplus are the same catalogue** — two Liquidity Services storefronts over one
backend, sharing `source_lot_id`, and `govliquidation.com` and `go-dove.com` are
that same app again. Reach past the deduped layer and you will show a user the
same forklift two or three times and count it twice.

## The calls

| Tool | Use it for |
|---|---|
| `us_auctions_search` | the main search — `keyword`, `state`, `near_zip` + `radius_miles`, `asset_type`, `segment`, `max_price` |
| `us_auctions_closing_soon` | urgency — what ends next, same filters |
| `us_auctions_sold_comps` | **what it actually sold for** — the valuation answer |
| `us_auction_events` | whole sales rather than lots (`min_lots`, `auction_house`) |
| `us_auction_houses` | who is selling, by state |
| `us_auctions_coverage` | sources, counts, freshness. Cite it |

## The trap that will produce a wrong number

`us_auctions_sold_comps` returns `matched_lots` **and** a `sampled` field, and
`matched_lots` is capped at 1,000. `sampled` is what tells you which you got.

Verified live 2026-09-02:

```
us_auctions_sold_comps({ keyword: "forklift", limit: 3 })
→ matched_lots: 1000, sampled: "most recent 1,000 matching sales",
  final_price: { min: 1, median: 1000, average: 2978, max: 61000 }

us_auctions_sold_comps({ keyword: "doosan b25s", limit: 2 })
→ matched_lots: 4, sampled: "all matching sales",
  final_price: { min: 1800, median: 1800, average: 1800, max: 1800 }
```

Two separate errors are waiting in the first one:

1. **`matched_lots: 1000` is a ceiling, not a population.** Never report it as
   "1,000 forklifts sold". Read `sampled` and quote it.
2. **Keyword matching pulls in accessories.** That `min: 1` is a *forklift cover*
   that sold for a dollar, and it is dragging the median. A broad keyword prices
   a category, not an asset.

So: **narrow to make-and-model before quoting a price**, and say how many
comps it rests on. Four comps at $1,800 is a more honest answer than a
thousand-row median of $1,000 — but say it is four.

## Time and distance are coarser than they look

- **Sale-level close dates are local calendar dates with no timezone**, stored
  that way deliberately: the local date is a fact, the exact instant is a guess.
  Do not parse them into a UTC timestamp and present an hour — an evening West
  Coast sale will shift a day.
- **`near_zip` distance is town-level**, computed from ZIP centroids. The
  response says so. Fine for "within 75 miles", wrong for "closest to me".

## A working recipe: "what is X worth, and can I buy one near me"

1. `us_auctions_sold_comps({ keyword: "<make model>" })` — as specific as you
   can. Check `sampled` and `matched_lots` before you believe the median.
2. If `matched_lots` is 1,000, narrow the keyword and run it again rather than
   reporting the capped median.
3. `us_auctions_search({ keyword, near_zip, radius_miles, max_price })` for
   what is currently biddable, priced off step 1.
4. `us_auctions_closing_soon({ ... })` for what needs a decision today.
5. Quote the comp count and the source freshness from
   `us_auctions_coverage({})` alongside any number you hand a person.

## What not to promise

Coverage is broad but not universal, and several large venues are deliberately
absent because they block automated access — Proxibid, BidSpotter,
LiveAuctioneers, Ritchie Bros and Purple Wave among them. Dealer-licensed
channels (Manheim) are closed entirely. If someone asks whether a specific
auction house is covered, check `us_auction_houses` rather than assuming, and say
plainly that absence from this store is not absence from the market.

Unsold lots are excluded from comps by design — a comp is a completed sale.
