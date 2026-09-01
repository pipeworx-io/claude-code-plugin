---
name: gtm-derived-signals
description: Compose primary-source data into derived account signals — the signal before the signal. Combine Form D filings, job postings, H-1B volume and vendor changelog activity into a scored target list instead of buying a static firmographic export. Use for account scoring, trigger-based targeting, territory prioritisation, and "which of these 300 accounts should I work this week".
---

# Derived signals

A firmographic field — headcount, industry, revenue band — describes a company's
steady state. It is the same next month. It cannot tell you *when*.

A derived signal is a change you compute yourself from primary sources: a filing
that appeared, a posting count that doubled, a changelog that went quiet. It is
the signal before the signal, because the underlying event lands in a public
record before it lands in a press release or a data vendor's refresh.

That is the whole method. The catalog's job is to hand you the primary records
cheaply and let you do the derivation; nobody else's scoring model has to be
right for yours to be.

## The four base signals available keylessly

| Signal | Tool | The change you compute |
|---|---|---|
| Capital raised | `form_d_recent_raises`, `form_d_issuer_history` | a new offering notice; a raise larger than the last |
| Hiring intent | `adzuna_search` | new postings for a role, count and recency |
| Headcount growth | `h1b_employer_sponsorship` | LCA filing volume year over year |
| Vendor/product motion | `vendor_signal_sweep` | changelog activity, and its absence |

`gtm-funding-signals`, `gtm-hiring-signals` and `gtm-company-resolution` cover
each source's arguments and traps. This skill is about combining them.

## Verified invocation

```
vendor_signal_sweep({})
```

Live result, 2026-09-01 (abridged):

```json
{
  "window_days": 7,
  "source": "Vendor changelog and release feeds (publisher-operated RSS/Atom and GitHub releases)",
  "vendors_tracked": 31,
  "vendors_with_activity": 7,
  "active": [
    { "vendor": "Cloudflare", "domain": "cloudflare.com", "changes": 6,
      "latest": { "published": "2026-08-26T00:00:00+00:00",
                  "title": "Workers AI - Z.ai GLM-5.3 Flash now available on Workers AI",
                  "url": "https://developers.cloudflare.com/changelog/post/2026-08-26-glm-5.3-flash-workers-ai/" } },
    { "vendor": "Vercel", "domain": "vercel.com", "changes": 6,
      "latest": { "published": "2026-08-26T00:00:00+00:00",
                  "title": "Python projects now support routing rules" } },
    { "vendor": "Github", "domain": "github.com", "changes": 5,
      "latest": { "published": "2026-08-26T12:18:59+00:00",
                  "title": "GitHub Apps can now access enterprise billing data" } }
  ],
  "quiet_vendors": ["auth0.com", "datadoghq.com", "docker.com", "elastic.co",
                    "hubspot.com", "mongodb.com", "okta.com", "openai.com",
                    "snowflake.com", "stripe.com", "supabase.com", "twilio.com", "..."]
}
```

**Two things to check before you use this, both observed on 2026-09-01:**

- The window appears fixed. Passing `window_days: 3` came back reporting
  `window_days: 7` with byte-identical content. Treat 7 days as the window.
- **Check `latest.published` before believing `quiet_vendors`.** On 2026-09-01
  every active item carried a `2026-08-26` timestamp. A vendor listed as quiet
  in a snapshot that has not moved in six days is not necessarily quiet — it may
  be a stale read. `quiet_vendors` is the more fragile half of this response and
  the half that is easiest to over-interpret.

`vendors_tracked: 31` is a fixed watchlist, not the whole software market. Do not
present an absence from it as evidence about a company.

## Composing a score

A shape that works, in rough order of signal strength for a technical B2B sale:

1. **Trigger** — did anything happen? A Form D in the last 90 days, or a first
   posting for the role you sell into. No trigger, no work.
2. **Capacity** — can they buy? `total_amount_sold` from the Form D,
   `lca_filings` count, `base_salary_usd.median` as a seniority proxy.
3. **Fit** — do they need it? `top_job_titles` from H-1B tells you what the org
   is building far more honestly than a self-reported industry code does.
4. **Recency** — `created` on postings, `filing_date` on notices. Weight the
   last 30 days heavily; a 6-month-old trigger is firmographics again.
5. **Identity** — resolve to `cik` / `lei` before you join any of it together.
   See `gtm-company-resolution`.

Write the score as code you can re-run. The point of deriving it yourself is
that you can change the weights when the readout says the list underperformed —
which you cannot do with a vendor's opaque intent score.

## Two honesty rules for anything you hand a human

**Print the denominator.** Every list-shaped tool here returns a count of what
matched alongside the rows it hydrated. `form_d_recent_raises` returns
`total_search_matches` (4,815 for a one-month window) next to `returned` (3).
A report that shows 3 rows and implies it swept the window is the most common
way this work goes wrong, and it is invisible to the reader.

**Carry the caveat with the number.** Form D returns an `interpretation` field
saying the notice is not proof a round closed. Adzuna returns
`salary_is_predicted`. H-1B returns `matched_employers` showing what it fuzzy
matched. Those fields exist because the underlying data has limits, and stripping
them out to make a tidy CSV is how a list becomes a liability.

## What is not here

There is no technographics source in the catalog — checked live,
`search_packs({ query: "detect what technology stack a website uses" })` returns
nothing that does it. If your score needs "runs Snowflake", that comes from
elsewhere. Say so rather than approximating it from job postings and calling it
technographics; a posting mentioning a technology is a weaker and differently
biased signal, and it should be labelled as what it is.
