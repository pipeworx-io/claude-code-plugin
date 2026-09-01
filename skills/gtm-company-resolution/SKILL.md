---
name: gtm-company-resolution
description: Resolve messy company names to stable registry identifiers — CIK, ticker, LEI, FIGI, Companies House number — and verify a domain you already have via RDAP and DNS. Use for list de-duplication, entity matching across sources, corporate-hierarchy questions, and before joining two enrichment sources on a name.
---

# Company resolution

The failure mode this exists to prevent: joining two lists on `company_name`.
"Revolut", "Revolut Ltd", "REVOLUT LIMITED" and "Revolut Technologies Inc" are
four strings, at least two legal entities, and one of them is dissolved. A join
on the string produces a list that is confidently wrong and looks fine.

Resolve to an **identifier** first, then join on that.

## The identifier you want, by situation

| You have | Reach for | You get |
|---|---|---|
| A US public company or ticker | `resolve_entity` | CIK, ticker, LEI, FIGI, subsidiary tree |
| A UK company | `companies_house_search_companies` → `companies_house_get_company` | company number, status, incorporation date, registered address |
| A US private issuer that has raised | `form_d_search_issuers` | CIK — see `gtm-funding-signals` |
| A domain, and you want to check it is real | `rdap_domain`, `dns_lookup_all` | registration date, registrar, nameservers, MX |

## Verified invocation

```
resolve_entity({ type: "company", value: "AAPL" })
```

Live result, 2026-09-01 (abridged):

```json
{
  "type": "company", "query": "AAPL", "resolved": true,
  "ids": { "cik": "0000320193", "ticker": "AAPL", "company_name": "Apple Inc.",
           "lei": "HWUPKR0MPOU8FGXBT394", "figi": "BBG000B9XRY4",
           "composite_figi": "BBG000B9XRY4", "share_class_figi": "BBG001S5N8V8" },
  "sources": { "cik": "sec-edgar", "lei": "gleif", "figi": "openfigi" },
  "unresolved": {},
  "relationships": {
    "parent": null, "ultimate_parent": null, "children_count": 8,
    "children": [
      { "lei": "5493004QI6E3PJNCEB09", "legal_name": "APPLE CHILE COMERCIAL LIMITADA" },
      { "lei": "549300YX4S1LLSMK2627", "legal_name": "APPLE ENERGY LLC" },
      { "lei": "5493006LHHR4CLPX4Q79", "legal_name": "BRAEBURN CAPITAL, INC." }
    ]
  }
}
```

The argument shape is `{ type, value }` — both required. `{ name: "..." }`
returns `Missing required parameters: type and value`.

The `relationships` block is the part list builders under-use. It is the LEI
parent/child graph, so it answers "is this account already covered by a parent
relationship" and "which subsidiaries share this buying center" without a data
vendor.

## Where it stops, honestly

**Name-based resolution of private companies mostly fails, and it tells you so.**

```
resolve_entity({ type: "company", value: "Databricks" })
  -> resolved: false
     ids: { lei: "984500E549A1FDC76F73" }
     unresolved.cik: "SEC EDGAR's API resolves by ticker or CIK only.
                      Pass a ticker or 10-digit CIK, or use ask_pipeworx
                      for a name-based search."

resolve_entity({ type: "company", value: "Snowflake" })
  -> resolved: false, ids: {}
     unresolved.lei: "GLEIF has 5 candidate(s) but none is an exact
                      legal-name match (match_mode=legal_name) — not
                      confident enough to assign one."
```

Both verified live 2026-09-01. Read those as a feature: the tool declines rather
than guessing, and it names the reason. A resolver that always returns something
is a resolver that silently mismatches your list. When you get
`resolved: false`, do not fall back to fuzzy string matching and call it
resolved — carry the failure into your output so a human can adjudicate.

**Name → website domain has no keyless answer in this catalog.** Checked live:
`search_packs({ query: "resolve a company name to its website domain" })`
returns packs that do not do it. If you need name→domain at scale, that is a
BYOK enrichment vendor path (`hunter`, `snov`, `findymail`, `pdl` — pass your
own key as `_apiKey`), or a web-search step. Say that plainly rather than
implying coverage.

## Verifying a domain you already have

Two keyless calls give you most of what a "company is real" check needs.

```
rdap_domain({ domain: "mailinator.com" })
```

Live result, 2026-09-01 (abridged):

```json
{
  "objectClassName": "domain",
  "ldhName": "MAILINATOR.COM",
  "events": [
    { "eventAction": "registration",  "eventDate": "2003-07-02T13:50:53Z" },
    { "eventAction": "expiration",    "eventDate": "2028-07-02T13:50:53Z" },
    { "eventAction": "last changed",  "eventDate": "2026-06-03T18:35:07Z" }
  ],
  "status": ["client delete prohibited", "client transfer prohibited", "..."],
  "entities": [ { "roles": ["registrar"], "handle": "146" } ]
}
```

The `registration` event is the useful one. A domain registered three weeks ago
behind a privacy registrar, on a list that claims a 200-person company, is a
signal — either bad data or a shell. Combine with `dns_lookup_all` (see
`gtm-email-hygiene`) for MX and TXT.

## Order of operations for de-duplicating a list

1. Normalize the string (case, legal suffixes, punctuation) — cheap, and it
   collapses the easy duplicates.
2. `resolve_entity` for anything that might be public. Join on `cik` or `lei`.
3. `companies_house_search_companies` for UK rows. **Check `status`** — the live
   Revolut query returns both `REVOLUT LTD` (active, incorporated 2013) and
   `REVOLUT LIMITED` (dissolved, incorporated 2010), plus 2,664 other matches
   including unrelated companies with fuzzy name overlap. Never take the first
   row. Print `total` next to your row count.
4. Anything still unresolved stays flagged. An unresolved row is a known unknown;
   a fuzzy-matched row is an unknown unknown.
