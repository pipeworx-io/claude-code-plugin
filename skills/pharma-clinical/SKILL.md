---
name: pharma-clinical
description: FDA drug regulatory data (approvals, labels, recalls, adverse events, warning letters), the ClinicalTrials.gov registry, and RxNorm drug-concept normalization, joined by the `pharma-intel` compound. Use for drug pipelines, safety-signal questions, approval/label history, sponsor trial portfolios, or any pharma/biotech question that has to come from primary regulatory or registry data rather than memory.
---

# Pharma and clinical data

Three keyless corpora, each with its own identity space:

- **ClinicalTrials.gov** (`clinicaltrials`) — what's being studied. Stable ID:
  `NCT#######`.
- **openFDA** (`openfda`) — what's been approved, labeled, recalled, or reported
  unsafe. **No stable company ID anywhere** — `sponsor_name` is exact-match free
  text.
- **RxNorm** (`rxnorm`) — canonical drug concepts (RxCUI) that let a brand name
  and its generic resolve to the same thing.

None of the three share an identifier. `pharma-intel`'s compounds join them by
**name matching**, not by ID — say so if you build a table across sources.

## The calls

| Tool | Use it for |
|---|---|
| `rxnorm_search` | name → RxCUI, split by term type (ingredient/brand/product) |
| `rxnorm_related` / `rxnorm_get_properties` / `rxnorm_ndc` | one RxCUI → related concepts, properties, NDC codes |
| `ct_search` | trials by condition/intervention/status/phase |
| `ct_sponsor_trials` | one company's trial portfolio |
| `ct_compare_sponsors` | head-to-head under identical filters |
| `ct_get_study` | one trial's full record, by NCT ID |
| `fda_drug_approvals` | Drugs@FDA application history |
| `fda_applications_by_cik` | every application form a company filed, resolved from its SEC ticker/CIK |
| `fda_drug_events` / `fda_event_counts` | FAERS adverse-event reports and reaction tallies |
| `fda_drug_labels` | current SPL label text |
| `fda_drug_recalls` / `fda_warning_letters` | enforcement actions |
| `pharma_drug_profile` | one drug, all four sources, one call |
| `pharma_safety_report`, `pharma_pipeline_scan`, `pharma_pipeline_catalysts`, `pharma_sponsor_diligence` | the other compound views |

## Trap: a FAERS count is reports, not patients, filed on nobody's authority

`fda_drug_events({ query: 'patient.drug.medicinalproduct:"OZEMPIC"' })`
verified live 2026-09-02: `total: 66161`. That is 66,161 **reports**, not
66,161 patients — one hospitalization commonly produces several linked
reports — and reporting is voluntary, at wildly uneven rates across consumers,
doctors, and manufacturers. There is no usable denominator behind that number.
**Never present a FAERS total as an incidence rate**, and a drug named in a
report is a temporal association, never a causal claim. None of this is in the
tool's schema; it has to be said every time you hand someone a FAERS number.

## Trap: a reaction filter needs FDA's exact term, and it won't tell you it missed

`fda_event_counts({ query: 'patient.drug.medicinalproduct:"semaglutide"', count_field: "patient.reaction.reactionmeddrapt.exact" })`
verified live 2026-09-02: top terms are `NAUSEA: 1075, VOMITING: 940,
DIARRHOEA: 646, OFF LABEL USE: 485, ...`. A reaction filter matches **one whole
MedDRA preferred term**, not a substring — searching `"NEUROPATHY"` misses
every case FDA filed as `"OPTIC ISCHAEMIC NEUROPATHY"`, because MedDRA's word
order is not the natural one and there is no partial match. Run
`fda_event_counts` first to see the terms FDA actually stores before you
filter on a guessed one.

## Trap: "Completed" on ClinicalTrials.gov is not "has an answer"

Neither `ct_search` nor `ct_get_study` says this anywhere in its schema:
`COMPLETED` means data collection ended, nothing about whether results exist.
Check the study record's `has_results` field separately — plenty of completed
trials never post here at all, publishing only in a journal instead. And a
status can already be stale when you read it: sponsors are only required to
refresh a study's record about once a year outside of major transitions, so a
"Recruiting" trial you pull today may have quietly stopped months ago.

## Trap: there is no drug-interaction tool, anywhere in this catalog

RxNorm's own interaction API (`rxnav` interactions) was discontinued by NLM in
January 2024 and nothing replaced it — there is no `rxnorm_interactions` tool
to find, so a caller checking the schema for one will find nothing rather than
an explanation. `pharma_safety_report` is the closest substitute and it is not
equivalent: it surfaces one drug's own FDA label Interactions section, not a
checked pairwise interaction. A drug absent from that text has not been
cleared as safe to combine — it just wasn't mentioned.

## Also verified live, worth citing whenever you use it

- `ct_sponsor_trials({ sponsor: "Merck Sharp & Dohme LLC" })` →
  `total_count: 2176`; the same call with
  `sponsor_match: "lead_or_collaborator"` → `total_count: 4282`. The default
  counts only the registered lead sponsor — say which mode you used.
- `fda_applications_by_cik({ cik_or_ticker: "LLY" })` → resolves to CIK
  `59478`, `total_applications: 143`, split across
  `sponsor_name_forms` `LILLY: 116`, `ELI LILLY AND CO: 22`,
  `ELI LILLY CO: 4`, `LILLY RES LABS: 1` — none of which is the SEC-registered
  name. Prefer this tool over guessing a `sponsor_name` string.
- `rxnorm_search({ name: "Ozempic" })` and `({ name: "Keytruda" })` both
  return the ingredient RxCUI directly in an `IN` concept group
  (semaglutide / pembrolizumab) alongside the brand and product groups — one
  call resolves brand → ingredient without a follow-up `rxnorm_related`.

## A working recipe: drug safety and pipeline snapshot for one name

1. `rxnorm_search({ name })` for the `IN`-group ingredient name — search
   openFDA's ingredient fields with that, not the brand, to catch reports
   filed under either spelling.
2. `fda_applications_by_cik({ cik_or_ticker })` for the sponsor's full
   application history if you have a ticker; otherwise `fda_drug_approvals`
   and read `openfda.brand_name` on the results for the filed spelling.
3. `fda_event_counts` with `count_field:
   "patient.reaction.reactionmeddrapt.exact"` to see real reaction terms
   before filtering `fda_drug_events` on one.
4. `ct_sponsor_trials({ sponsor })` or `ct_search({ query: name })` for what's
   still in development — state which `sponsor_match` you used.
5. Or skip 1-4 for one drug: `pharma_drug_profile({ drug_name: name })` runs
   the equivalent join in one call — verified live 2026-09-02 on
   `{ drug_name: "ozempic" }`, returning populated `rxnorm`, `fda_approvals`,
   and safety branches together.

## What not to promise

- `fda_warning_letters` recipient search pages a full-text superset up to 600
  candidates; past that it returns `scan_complete: false` and a count that is
  a floor, not a total.
- Drugs@FDA (`fda_drug_approvals`) has no indication/disease field at all —
  "drugs approved to treat X" has to go through `fda_drug_labels` instead.
- ClinicalTrials.gov is US-registered but includes foreign studies with any US
  site; a purely non-US trial may not be here at all.
