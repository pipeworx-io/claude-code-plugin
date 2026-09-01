---
name: gtm-hiring-signals
description: Find companies hiring for roles that imply they need what you sell, using live job-posting data (Adzuna) and US H-1B LCA filings. Use for hiring-signal targeting, "who is hiring X right now", headcount and salary benchmarking, or building a list from job-market activity rather than firmographics.
---

# Hiring signals

The premise: a job posting is a budget decision that already happened. If a
company is hiring three data engineers, it has a data problem and money against
it. That is a better trigger than a firmographic filter.

## Read this before you pick a tool — market-facing vs your-own-ATS

The catalog contains both, and they are not interchangeable. Getting this wrong
produces a list that looks right and is empty of the companies you wanted.

**Market-facing (scans the job market, any company):**

- `adzuna_search` — live aggregated postings, per country, with structured
  location and salary. This is the list-building tool.
- `adzuna_history` — monthly salary series for a role/geo. Trend, not rows.
- `adzuna_categories` — the category taxonomy, for clean faceting.
- `h1b_employer_sponsorship`, `h1b_salary`, `h1b_top_sponsors` — US Labor
  Condition Applications. Employer-level, disclosed, and quantitative.

**Your own ATS (reads YOUR account, not the market):**

- `greenhouse_*` and `ashby_*` — `greenhouse_list_candidates`,
  `ashby_list_applications`, `*_get_job`. These are recruiting-ops tools for a
  company reading its own pipeline with its own credentials.

**Do not build a market-scan on greenhouse or ashby, and do not describe them as
hiring-signal coverage.** They cannot tell you which companies are hiring. The
market-facing answer is Adzuna plus H-1B, and that is what you should say when
someone asks about coverage.

## Verified invocation

```
adzuna_search({ country: "us", what: "solutions engineer", where: "san francisco" })
```

Live result, 2026-09-01 (one row of many):

```json
{
  "results": [
    {
      "title": "Solutions Engineer",
      "company": { "display_name": "Jacobs" },
      "location": { "display_name": "SoMa, San Francisco",
                    "area": ["US", "California", "San Francisco", "SoMa"] },
      "created": "2026-08-20T19:33:11Z",
      "salary_min": 162061.84,
      "salary_max": 162061.84,
      "salary_is_predicted": "1",
      "category": { "tag": "engineering-jobs", "label": "Engineering Jobs" },
      "redirect_url": "https://www.adzuna.com/land/ad/5850177047?..."
    }
  ]
}
```

**`salary_is_predicted` matters.** `"1"` means Adzuna modelled that number from
the posting text; the employer did not publish it. In the row above `salary_min`
and `salary_max` are identical, which is the shape a prediction takes. Never put
a predicted salary in front of a customer as a disclosed figure. Filter on
`salary_is_predicted === "0"` when the number has to be real.

`created` is the posting date — the freshness of the signal, and the thing you
sort by when you want "started hiring this week" rather than "has a stale req".

## H-1B: the quantitative half

```
h1b_employer_sponsorship({ employer: "Databricks" })
```

Live result, 2026-09-01:

```json
{
  "employer": "Databricks",
  "year": 2025,
  "sponsors_h1b": true,
  "lca_filings": 381,
  "matched_employers": ["DATABRICKS INC"],
  "base_salary_usd": { "min": 103646, "median": 160805,
                       "average": 167589, "max": 325000 },
  "top_job_titles": [
    { "value": "SOFTWARE ENGINEER", "count": 106 },
    { "value": "SR. SOFTWARE ENGINEER", "count": 52 },
    { "value": "SOLUTIONS ARCHITECT", "count": 21 },
    { "value": "SR. SOLUTIONS ARCHITECT", "count": 19 }
  ]
}
```

`base_salary_usd` here is **disclosed**, not predicted — LCA filings carry a
legally attested wage. That makes H-1B the better source for compensation
benchmarking and Adzuna the better source for freshness and breadth.

Check `matched_employers` on every call. It is the tool telling you which legal
entity names it fuzzy-matched, and it is where a wrong answer becomes visible:
if you asked for a company and got a staffing agency, you will see it there.

**Coverage limits, say them:** H-1B is US-only, employer-level, annual, and
covers only sponsoring employers. A company with `sponsors_h1b: false` is not
"not hiring" — it is not sponsoring. Adzuna is per-country (pass `country`) and
aggregates job boards, so it undercounts companies that only post on their own
careers page.

## Composing the two

The pair answers "hiring, and at what level" better than either alone:

1. `adzuna_search` for the role and geography you sell into → candidate
   companies, with posting dates.
2. `h1b_employer_sponsorship` per surviving company → filing volume as a
   headcount-growth proxy, disclosed salary bands as a seniority read, and
   `top_job_titles` as a read on what the org is actually building.
3. Score on: recency of postings, count of postings for the same role,
   `lca_filings` year over year. Volume plus recency beats either alone.

For the wider composition pattern see `gtm-derived-signals`.
