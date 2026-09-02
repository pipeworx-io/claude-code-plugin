---
name: trade-and-tariffs
description: International and US bilateral trade flows, US customs duty revenue, and US import-compliance precedent (CBP rulings, antidumping/countervailing orders) — for "how much did the US import from China", "what does Germany export to Brazil", "is there an AD/CVD order on this product from this country", or any trade-balance / tariff-context question. Not a live tariff-rate calculator — that is the separate `hts` pack.
---

# Trade flows, customs revenue, and import compliance

Four corpora, not one, and they answer different questions:

- **UN Comtrade** (`comtrade_*`) — every reporting country's bilateral flows,
  goods only, HS-coded. The global source; annual, lags ~6-12 months.
- **US Census** (`census_*`) — the US side specifically, monthly, with its
  own Schedule C country codes (not ISO — pass `country` as a name and it
  resolves for you).
- **US Treasury** (`treasury_*`) — customs duty *revenue*, not trade value;
  AD/CVD collections and fees are mixed into the same "Customs Duties" line.
- **Trade compliance** (`cross_*`, `adcvd_*`) — CBP classification precedent
  and AD/CVD legal status, not trade volume.

`trade-intel`'s compounds (`trade_bilateral_analysis`, `trade_country_profile`,
`trade_macro_dashboard`) chain the above and degrade per-source — read each
subcall's `reason` before trusting a null.

## The calls

| Tool | Use it for |
|---|---|
| `comtrade_trade_data` | one reporter/partner/year/flow — the base bilateral query |
| `comtrade_top_partners` | who a country trades most with, ranked, `TOTAL` or one HS code |
| `comtrade_top_commodities` | what two countries actually trade, ranked by value |
| `comtrade_country_codes` | UN numeric codes (US=842, not ISO 840) |
| `census_imports` / `census_exports` | US side by HS code and/or country, monthly |
| `census_trade_balance` | US bilateral balance with one country |
| `census_trade_trends` | monthly US trend with one country (all-countries hangs — see below) |
| `treasury_customs_revenue` | monthly Customs Duties collections, gross/refund/net |
| `treasury_exchange_rates` | quarterly federal-accounting rates, not market FX |
| `cross_search` / `cross_ruling` | CBP classification precedent by product or ruling number |
| `adcvd_orders` / `adcvd_case` | AD/CVD order status by product/country; full case history |

## Trap 1: reporter and partner report the same flow differently — verified live

China's exports to the US and the US's imports from China describe the
identical physical shipments, filed by two different customs agencies.
Verified live 2026-09-02, calendar year 2023:

```
comtrade_top_partners({ reporter_code: "842", year: "2023", flow: "M", limit: 5 })
→ China: trade_value_usd 448,017,376,384

comtrade_top_partners({ reporter_code: "156", year: "2023", flow: "X", limit: 5 })
→ USA:   trade_value_usd 501,220,722,660
```

**A $53.2B gap (10.6%) on the same year, same flow, both official.** Neither
number is wrong — valuation basis, transshipment/re-export handling (Hong
Kong strips out separately), and timing of customs clearance differ by
reporter. Never present one side's figure as *the* number; if precision
matters, pull both and say so.

## Trap 2: HS code depth changes what the number means — verified live

`comtrade_top_commodities`'s HS2/HS4/HS6 aggregation levels are documented in
its schema; what is not documented is how much a truncated code silently
folds in. Verified live, US imports from China, 2023:

```
comtrade_trade_data({ reporter_code:"842", partner_code:"156", year:"2023", hs_code:"84", flow:"M" })
→ 85,889,376,499   (chapter 84: ALL machinery)

comtrade_trade_data({ reporter_code:"842", partner_code:"156", year:"2023", hs_code:"8471", flow:"M" })
→ 39,640,518,148   (heading 8471: computers/data-processing machines only)
```

Computers are 46% of "machinery" — the other 54% ($46.2B) is pumps, turbines,
dishwashers, and everything else in chapter 84. A 2-digit code is a category,
not a product; never answer a specific-product question with a chapter total.

**Related, also verified:** omitting `hs_code` from `comtrade_trade_data`
does *not* return an aggregate — it returns individual HS6 line items, capped
at 500 rows, that must be summed yourself and may undercount a reporter with
more than 500 traded lines. To get the real total, pass `hs_code: "TOTAL"`
(undocumented in the schema, but it works — confirmed it returns the
identical $448,017,376,384 as the `comtrade_top_partners` total above).

## Trap 3: "trade value" is customs value, not landed cost

Census publishes two different import value fields for the same shipment:
`GEN_VAL_MO` ("General Imports, Total Value") and a separate `GEN_CIF_MO`
("General Imports, CIF Value") — confirmed against Census's own variable
dictionary (`api.census.gov/.../imports/hs/variables.json`). `census_imports`/
`census_exports` request only `GEN_VAL_MO` — customs (FAS-basis) value,
excluding international freight and insurance. A question about landed cost
(freight + insurance included) is understated by this number, and there is no
argument to switch basis — it isn't wired in. Comtrade has the same split
(its README calls it CIF vs FOB); say which basis you're quoting.

## Verified invocations

All run 2026-09-02 through `scripts/pwcall.sh`, arguments exactly as shown.

```
adcvd_orders({ country: "China", product: "steel" })
```
Five order groups, all `status: "in_place"` — one (Non-Oriented Electrical
Steel) carries eight case numbers across five countries, so one "product"
query can return a multi-country order.

```
treasury_customs_revenue({ limit: 3 })
→ record_date 2026-07-31: current_month_gross 24,834,855,900.89,
  current_month_refund 33,381,297,807.86, current_month_net -8,546,441,906.97,
  fiscal_year_net 154,472,685,103.20
```
July 2026's **monthly net is negative** — refunds exceeded that month's gross
collections — while fiscal-year-to-date is a $154B surplus. A monthly
snapshot alone can read as "tariff revenue went negative"; check
`fiscal_year_net` before concluding a trend reversed.

## A working recipe: "what does the US actually pay to import X from Y"

1. `comtrade_top_commodities({ reporter_code, partner_code, flow:"M", hs_level:4 })`
   to find the right HS heading, not chapter (Trap 2).
2. `comtrade_trade_data({ ..., hs_code: "<that heading>" })` for the reporter
   figure, then the mirror call with reporter/partner swapped and `flow`
   flipped for the partner's figure (Trap 1) — quote both if they diverge.
3. `cross_search({ query: "<product>" })` for classification precedent, and
   `adcvd_orders({ product, country })` for any trade-remedy duty — read
   `scope_note` before ruling a product "not covered."
4. Name the value basis (Trap 3) if the question is about landed cost.

## What not to promise

Comtrade lags 6-12 months and some reporters skip years (Cuba, Iran, North
Korea) — a partner-side mirror query can fill a gap. `census_trade_trends`
refuses an all-countries query outright rather than hanging; use
`census_imports`/`census_exports` for worldwide totals instead. `adcvd_orders`
returns recent Federal Register *activity*, not a census of orders in place —
absence means no recent notice, not no order. This corpus is goods and duty
revenue only: no live tariff-rate calculator (that's `hts`), no services
trade, no shipment-level tracking.
