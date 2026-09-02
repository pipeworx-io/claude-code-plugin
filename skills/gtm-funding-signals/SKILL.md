---
name: gtm-funding-signals
description: Build funding-based target lists from SEC Form D exempt-offering notices — recently raised private companies, PE/VC fund formation, repeat founders, amendment chains. Use when asked for seed-funded or PE-funded company lists, "who raised recently", private financing diligence, or a Crunchbase/PitchBook-shaped question that has to come from a primary source.
---

# Funding signals from Form D

Form D is the notice a US private issuer files with the SEC when it sells
securities under a Regulation D exemption. It is filed within 15 days of the
first sale. That is usually **weeks or months before the round is announced** —
which is why it is a signal rather than news.

Coverage is keyless and free. There is no seat, no credit, and no per-row cost.

## What Form D is and is not

Say this out loud in any output you build from it, because a list that implies
more than the data supports is worse than no list:

- A Form D is a **filer-supplied notice of an exempt offering**. It is not proof
  that a financing closed.
- `total_amount_sold` is the amount reported **as of that filing**. An amendment
  (`D/A`) may restate the same offering. **Never sum a chain** — you will
  double-count.
- Coverage is US Reg D only. No foreign rounds, no priced rounds that skipped
  Reg D, no debt that was not a security sale.
- A large share of filings are **pooled investment funds**, not operating
  companies. If you want companies, filter `industry_group`.

The tool returns an `interpretation` field carrying that caveat. Keep it
attached to the data when you hand a list to a human.

## The calls

| Tool | Use it for |
|---|---|
| `form_d_recent_raises` | "who raised recently" — the list-building entry point |
| `form_d_search_issuers` | free-text search by issuer, fund, or filing text |
| `form_d_issuer_history` | one issuer's full filing history (needs `cik`) |
| `form_d_amendment_chains` | de-duplicate originals from amendments (needs `cik`) |
| `form_d_related_person_search` | map repeat founders and fund managers by name |
| `form_d_offering_detail` | one filing by accession number, fully normalized |

### Arguments that actually work

`form_d_recent_raises` takes `since`, `until`, `days`, `industry`,
`minimum_sold` (alias `min_amount`), `include_amendments`, `limit` (1-10).

Amount filtering works: `minimum_sold` / `min_amount` is a floor on
`total_amount_sold` in USD. `days` looks back that many days from `until`; an
explicit `since` wins over it. Anything else is **rejected** with the list of
accepted arguments — a misspelled argument errors rather than being dropped.

**The trap that remains — read your denominators.** Filters are applied to
hydrated filings, and only the newest slice of the window is hydrated. Three
counts come back and they mean different things:

- `total_search_matches` — every Form D in the window, before any filter.
- `scanned` — how many of those were actually inspected (up to 60 when
  filtering).
- `matched_in_scan` — how many of the scanned ones passed your filter.

So `returned: 5` out of `total_search_matches: 4829` is **not** "only 5 raises
over $10M exist" — it is "5 of the 60 newest notices cleared the floor."
Narrow `since`/`until` and page through rather than reporting a short list as
scarcity, and quote `scanned` alongside any count you give a user.

## Verified invocation

```
form_d_recent_raises({ since: "2026-08-01", industry: "Technology", limit: 3 })
```

Live result, 2026-09-01:

```json
{
  "window": { "since": "2026-08-01", "until": "2026-09-01" },
  "total_search_matches": 4815,
  "returned": 3,
  "interpretation": "Form D is a filer-supplied notice of an exempt offering, not independent verification that a financing closed. Amount sold is the amount reported as of this filing; amendments may restate the same offering.",
  "offerings": [
    {
      "issuer_name": "Neo Agent Inc.",
      "industry_group": "Other Technology",
      "total_amount_sold": 2744996,
      "filing_date": "...",
      "accession_number": "...",
      "cik": "...",
      "related_persons": [ { "name": "...", "relationships": ["Executive Officer", "Director"] } ]
    }
  ]
}
```

Note what comes back per row that a lead database charges for: `cik` (a stable
join key), `jurisdiction`, `year_of_incorporation`, `revenue_range`,
`minimum_investment`, `investors_already_invested`, and named executives with
their relationships. That is a targetable record, not just a name.

## A working recipe: "seed-funded companies, last 24 months"

1. Walk `since`/`until` in monthly slices — a 24-month single call is one
   window and `limit` is 10, so you would see 10 of tens of thousands.
2. `industry` to drop `Pooled Investment Fund` rows if you want operating
   companies.
3. Filter `total_amount_sold` client-side to your band. Seed shape is roughly
   $500k-$5M with `investors_already_invested` in single or low double digits.
4. De-duplicate on `cik`, not on `issuer_name` — series LLCs and fund vehicles
   reuse names.
5. Drop `is_amendment: true` rows, or resolve them with
   `form_d_amendment_chains({ cik })` and keep the latest per chain.
6. Enrich the survivors from other packs (see `gtm-derived-signals`).

## Related-person search is the underrated one

`form_d_related_person_search({ person: "<name>", since: "2022-01-01" })`
returns every notice whose parsed related-person list matches. That maps repeat
founders and the funds that follow them — a relationship graph out of primary
filings, for free. Relationships are filer-supplied, so treat them as claims.

The argument is `person`, not `query` or `name`. Verified live:
`{ person: "Marc Andreessen", since: "2022-01-01", limit: 2 }` returns
`Andreessen Horowitz LSV Fund III, L.P.` among others. `form_d_search_issuers`
is the one that takes `query`.
