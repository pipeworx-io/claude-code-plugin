---
name: legal-research
description: US federal case law and litigation dockets from primary sources — find a case by name or citation, read the opinion, pull the docket sheet, fetch the filed PDFs. Use for case lookup, a party's litigation history, "what was actually filed in X", or any legal claim that has to come from the record rather than from memory.
---

# Legal research from the primary record

Two different corpora sit behind these tools, they fail in different ways, and
conflating them is how a confident wrong answer gets built.

- **Opinions** — 10M+ decision clusters. Retrieval by citation or case name is
  mirrored locally: fast, keyless, reliable.
- **Dockets and filings** — the federal litigation record via RECAP. Keyless,
  but **crowd-sourced**: a document exists in it only because somebody once paid
  PACER for that PDF. Partial by construction.

There is **no keyless coverage of state statutes or administrative codes.** If
someone asks for a state law, say so rather than reaching for a federal tool
that will answer a different question.

## The calls

| Tool | Use it for |
|---|---|
| `lookup_citation` | a citation you already have → the case. Exact, mirrored, fast |
| `find_case` | a case by name when you don't have the cite |
| `search_opinions` | subject-matter search across decisions |
| `search_case_law` | broader opinion search when `search_opinions` is too narrow |
| `list_case_opinions` | every opinion in one cluster (needs `cluster_id`) |
| `get_opinion` | one opinion's full text (needs `id`) |
| `court_listener_search_dockets` | find a docket by party, case name, or subject |
| `list_docket_filings` | the docket sheet — every entry (needs `docket_id`) |
| `get_docket_filing` | one filing (needs `document_id`) |
| `recap_filings` | the actual PDFs for a docket, from the Internet Archive mirror |

### The name collision, which will bite you first

**`search_dockets` is not a callable name.** It collides with the
regulations.gov pack, so both are exposed namespaced:
`court_listener_search_dockets` and `regulations_gov_search_dockets`. Every
*other* tool in this table is exposed under its bare name. Calling
`search_dockets` returns `Unknown tool`, and calling
`court_listener_list_docket_filings` does too — the namespacing is per-collision,
not per-pack.

## Read your denominators — a missing PDF is not a missing filing

`list_docket_filings` returns two counts that mean different things:

- `total_filings` — entries on the docket sheet.
- `pdfs_available` — how many of those have a document you can actually read.

Verified live 2026-09-02 on docket 71886217: `total_filings: 18`,
**`pdfs_available: 1`**. Seventeen of those entries are real filings that
happened; nobody has bought the PDF. Every row carries `pdf_available` for
exactly this reason.

So never write "only one document was filed." Write "18 docket entries, 1 with a
retrievable PDF." The docket text is the record; the PDFs are a sample of it.

## Reliability: the docket path goes upstream, the opinion path does not

Citation lookup is served from our mirror. The docket endpoints hit
CourtListener live, against a low anonymous throttle, and **they time out under
normal conditions**. Measured this session: three docket-path calls, two of them
returned a 25s timeout, and both succeeded on retry with identical arguments.

**Retry once before reporting anything as down or empty.** A timeout here is not
evidence about the case — it is evidence about the connection.

## Verified invocations

All run 2026-09-02, arguments exactly as shown.

```
lookup_citation({ citation: "410 U.S. 113" })
```
```json
{ "resolved_by": "exact_citation",
  "cases": [{ "cluster_id": 108713, "case_name": "Roe v. Wade",
              "date_filed": "1973-01-22", "citation_count": 5581 }] }
```

```
court_listener_search_dockets({ query: "antitrust merger", court: "nysd" })
```
```json
{ "total": 1911, "returned": 20,
  "dockets": [{ "docket_id": 66692954,
                "caseName": "Funicular Funds, LP v. Pioneer Merger Corp.",
                "docketNumber": "1:22-cv-10986", "dateFiled": "2022-12-30" }] }
```

```
recap_filings({ docket_id: 73433096 })
```
```json
{ "docket": { "case_name": "V.O.S. Selections, Inc. v. Trump", "court": "cafc" },
  "archive_identifier": "gov.uscourts.cafc.24456",
  "documents": [{ "docket_entry": 1, "size_bytes": 276572,
                  "url": "https://archive.org/download/gov.uscourts.cafc.24456/gov.uscourts.cafc.24456.1.1.pdf" }] }
```

`recap_filings` addresses the Internet Archive mirror by a **derivable**
identifier, `gov.uscourts.<court>.<case_id>` — no lookup table, no key, and the
URLs it returns are directly fetchable.

## A working recipe: "what has this party been sued over, and what was filed"

1. `court_listener_search_dockets({ query: "<party name>" })` — add `court` to
   narrow. Take `docket_id` from the rows.
2. `list_docket_filings({ docket_id, limit })` for the docket sheet. Read
   `total_filings` and `pdfs_available` before you characterise it.
3. `recap_filings({ docket_id })` for the archive PDFs, or follow `pdf_url` on
   an individual filing from step 2.
4. Retry any step that times out **once** before concluding anything.

For the opinion side, go `lookup_citation` or `find_case` →
`list_case_opinions({ cluster_id })` → `get_opinion({ id })`.

## Relevance: cite counts beat text matches

Every cluster carries `citation_count`. In law that is the strongest available
relevance signal — a case cited 5,581 times outranks a closer textual match
nobody has ever cited. When you rank or summarise results, rank on it and say
you did. `precedential_status: "Published"` is a much weaker filter: over half
the corpus has never been cited by anything, and most of that is unpublished
orders and procedural dispositions.

## What to say about coverage, unprompted

RECAP's docket coverage is partial and the tools do not measure it for you. If
you build a list or a litigation history from it, attach the caveat: *this is
the crowd-sourced federal record; absence here is not absence of a case.* A
list that implies completeness is worse than a shorter honest one.

`pacer` and `docket-alarm` reach the paid sources and are bring-your-own-key —
reach for them only when the caller has said they have credentials.
