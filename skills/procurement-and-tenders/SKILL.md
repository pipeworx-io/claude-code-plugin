---
name: procurement-and-tenders
description: Government procurement — tenders, contract awards, and who won them — across the EU (TED), UK, Switzerland, Ukraine, Moldova, US federal (usaspending, DoD), US city/county bid portals, and a 19-country Open Contracting (OCDS) mirror, plus GLEIF for resolving a winning company to its legal entity. Use for "who won this government contract", "what is X bidding on", "public spending on Y in country Z", or any tender/award question that has to come from an official notice rather than memory.
---

# Public procurement and tenders

A tender notice, a contract-award notice, and a signed contract are three
different objects, and no source in this corpus reliably carries a caller from
the first to the third. Which one a given tool returns — and how completely —
varies by country, because every source here implements the Open Contracting
Data Standard (OCDS) or its own schema **partially**, not by a shared contract.
Never assume a field's absence means zero, and never assume one jurisdiction's
conventions (currency, lifecycle, entity identity) hold for another.

## The calls

| Tool | Use it for |
|---|---|
| `ted_eu_search_notices` / `ted_eu_get_notice` / `cpv_lookup` | EU tender + award notices, all 27 member states, above-threshold |
| `ted_search_awards` / `ted_supplier_history` / `ted_buyer_profile` | who won — by supplier or buyer, TED award notices only |
| `uk_contracts_search_notices` / `recent_notices` / `find_a_tender_recent` | UK — Contracts Finder (below-threshold+awarded) and Find a Tender (above) |
| `simap_search` / `simap_recent` / `simap_project` | Switzerland — non-EU, not on TED, CHF |
| `prozorro_search_tenders` / `prozorro_get_tender` / `prozorro_search_organizations` | Ukraine — UAH, tender-stage data only |
| `moldova_recent_tenders` / `moldova_get_tender` / `moldova_search_tenders` | Moldova (MTender) — MDL, tender-stage data only |
| `oc_tender_search` / `oc_recent` / `oc_process_history` / `oc_coverage` | hosted OCDS mirror, 19 countries — call `oc_coverage` first |
| `usa_award_search` / `usa_recipient_profile` / `usa_spending_by_agency` | US federal contract awards — USD, FPDS-sourced |
| `dod_awards_recent` / `dod_awards_search` / `dod_awards_top` | US defense awards ≥$7.5M, same-day — leads usaspending by weeks |
| `open_bids_search` / `gov_bids_jurisdictions` | US city/county **open bids** — pre-award, not contracts |
| `search_lei` / `get_lei_relationships` / `lei_hierarchy_tree` | resolve a winning company name to its legal entity + ownership tree |

## Jurisdiction coverage — say this plainly, unprompted

Covered directly: all 27 EU states above-threshold (TED), UK, Switzerland,
Ukraine, Moldova, US federal + DoD awards, ~18 US city/county bid portals
(open bids only). Covered via the hosted OCDS mirror, verified live
2026-09-02 (`oc_coverage`: 843,165 processes, 19 countries): Albania,
Argentina, Croatia, Dominican Republic, Ghana, Guatemala, Honduras, Italy,
Kenya, Kosovo, Liberia, Mexico (Oaxaca), Nigeria, Peru, Rwanda, Tanzania,
Thailand (Bangkok), Uruguay, Zambia.

**Not covered — the failure mode to guard against:** most of Asia (India,
China, Japan, SE Asia), the Gulf states, most of Latin America and Africa
outside the lists above. Within the EU, **below-threshold national tenders in
most member states are unreachable** — Belgium's `enot.publicprocurement.be`
firewalls non-EU/datacenter IPs at the TCP level (confirmed), an
infrastructure wall, not a gap we haven't built past. A "global procurement"
answer from this catalog is real for the jurisdictions above and silent
everywhere else.

## Trap 1: OCDS is a standard; population is not — verified live 2026-09-02

```
oc_tender_search({ country: "Kenya", limit: 5 })
→ 5/5 rows: no value_amount, no value_currency, no status, no award_* field

oc_tender_search({ country: "Italy", limit: 5 })
→ 5/5 have value_amount+value_currency (EUR), 3/5 have award_date/amount/
  supplier. One shows the split directly: value_amount 3,582,605.90 vs
  award_amount 3,445,838.29 (negotiated down after tender).
```

Kenya's PPRA publishes no monetary field at all; Italy's ANAC publishes both
estimate and settled award. The pack correctly *omits* an unset field rather
than showing 0 (no `value_amount: 0` anywhere in the Kenya rows) — but a
caller who doesn't check for the field's presence before summing values will
silently drop Kenya to zero instead of "unknown." Call `oc_coverage` first.

## Trap 2: a "value" is what was estimated, not what was paid

Neither `moldova-tenders` nor `prozorro` ever reads an OCDS `awards` array —
confirmed by reading both packs' source, no `awards`/`contracts` reference in
either file. Every value returned is the tender's own declared amount, at
whatever stage the record is in — verified live:

```
prozorro_search_tenders({ tenderer: "00131305" })
→ first row: status "cancelled", value.amount 1,193,300 UAH

moldova_recent_tenders({ limit: 3 })
→ one row: status "complete", value: null, title: null, buyer: null
```

A cancelled Ukrainian tender carries a fully-populated value — nothing was
ever spent, but it looks identical to a real transaction. A "complete"
Moldovan record can come back entirely null, because completion doesn't
guarantee the compiled release still carries the fields this pack reads.
Never report a `value` from either pack as "awarded" or "spent."

## Trap 3: GLEIF resolves one entity; a name search resolves however many share it

`search_lei({ query: "Siemens Mobility" })` returns 42 distinct active LEIs
worldwide sharing that name, one per national subsidiary. `get_lei_relationships`
on the Belgian one (`529900R6QFNA89TXEZ72`) confirms its direct AND ultimate
parent is Siemens Aktiengesellschaft (`W38RGI023J3WT1HWRP32`) — a different LEI.
Meanwhile `ted_supplier_history({ supplier: "Siemens Mobility" })` reports
`matched_on: "winner-name~... every entity whose published name contains it, in
any member state"` — 647 awards summed across however many of those 42 legal
entities won an EU contract under that string. **"How much did Siemens
Mobility win" gets a different, non-overlapping answer depending on whether
you mean the string or one LEI** — TED aggregates by name across entities
GLEIF would call legally distinct, and GLEIF resolves exactly one per call,
never the group. Neither schema says this; it only shows up by running both.

## Trap 4: two of the most obvious tool names here are namespaced

`search_notices` and `get_notice` are not callable under those names —
`pwcall.sh names search_notices` / `names get_notice` show they collide
across `ted-eu`, `uk-contracts`, and `france-boamp` (gazette), resolving to
`ted_eu_search_notices` / `uk_contracts_search_notices` / `ted_eu_get_notice`
/ `uk_contracts_get_notice` on the live gateway. Both packs' own source lists
the bare name in their `tools` array, so reading pack code rather than the
live tool list produces a call that doesn't exist. Every other tool named
here was checked the same way and resolves bare.

Every invocation above was run 2026-09-02 through `scripts/pwcall.sh`, exactly
as shown, with the real (non-empty) response — no example in this skill is
inferred from a schema.

## A working recipe: "who won this contract, and who really owns them"

1. `ted_eu_search_notices` / `ted_search_awards` (or the national pack for a
   non-EU country) — read the notice's own `currency`, never assume EUR.
2. `search_lei({ query: winner_name })` — check `total`; more than one hit
   means more than one legal entity shares that name, so pick one.
3. `get_lei_relationships` (one hop) or `lei_hierarchy_tree` (multi-level) on
   the chosen LEI — state which entity, not just which name, the answer covers.
4. `ted_supplier_history({ supplier: name })` for a name-based total, and say
   explicitly it's name-matched, not entity-matched (Trap 3), if you also
   cite a GLEIF-anchored number in the same answer.

## What not to promise

No tool here converts currency — TED is "normally EUR but not all," Ukraine
is UAH, Moldova is MDL, Switzerland is CHF, US is USD; a sum across
jurisdictions without carrying each currency is noise, not a number. No
source guarantees the tender → award → contract chain completes: an open
tender may never show an award, and `open_bids_search` never becomes a
contract record at all (that's `usa_award_search` / a national award tool).
GLEIF's ISIN/BIC mapping lags issuance, and a valid LEI with no relationship
record is unmapped, not parentless.
