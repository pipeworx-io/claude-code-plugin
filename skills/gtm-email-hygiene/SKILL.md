---
name: gtm-email-hygiene
description: Check whether a domain can actually receive mail and who runs it, using keyless DNS and RDAP — MX presence, SPF/DMARC posture, mail provider, domain age. Use before sending to a purchased list, when scrubbing bounces, when asked whether a domain is disposable, or when list quality is suspect.
---

# Email and domain hygiene

Two questions get conflated and they have different answers:

1. **Is this domain real, and can it receive mail?** Answerable keylessly, right
   now, from DNS and RDAP. That is this skill.
2. **Does this specific mailbox exist?** SMTP-level verification. Requires a
   vendor. See the BYOK section at the bottom.

Do the free one first. It is fast, it costs nothing per row, and on a bought
list it removes a large fraction of the garbage before you spend a verification
credit on it.

## The keyless checks

```
dns_lookup_all({ domain: "<domain>" })
```

Live result for `mailinator.com`, 2026-09-01 (abridged):

```json
{
  "domain": "mailinator.com",
  "records": {
    "A":  [ { "value": "...", "ttl_seconds": 300 } ],
    "MX": [ { "value": "1 mail.mailinator.com.",  "ttl_seconds": 300 },
            { "value": "1 mail2.mailinator.com.", "ttl_seconds": 300 } ],
    "TXT":[ { "value": "v=spf1 -all", "ttl_seconds": 300 } ],
    "NS": [ "..." ],
    "AAAA": [ "..." ],
    "CNAME": []
  }
}
```

Returns `A`, `AAAA`, `MX`, `NS`, `TXT`, `CNAME` in one call.

### What each record actually tells you

- **No `MX` records at all** → the domain cannot receive mail. Everything you
  send to it bounces. This is the single highest-yield free filter on a bought
  list, and it needs no vendor.
- **`MX` present** → the domain accepts mail. **That is all it means.**
  `mailinator.com` — a disposable-mail service — has MX records. MX presence is
  necessary, not sufficient, and anyone telling you MX presence proves a domain
  is legitimate is wrong.
- **Who the MX points at** is the real read. `aspmx.l.google.com` or
  `*.mail.protection.outlook.com` means Google Workspace or Microsoft 365 — a
  company paying for business email. Self-hosted MX on the domain's own hostname
  (`1 mail.mailinator.com`, as above) is neutral for a large operator and
  suspicious for a 12-person company you have never heard of.
- **`TXT` → SPF.** `v=spf1 -all` on `mailinator.com` is a **hard fail for every
  sender**: the domain declares that nothing is authorized to send as it. A
  domain that receives mail but has publicly disavowed all outbound is not a
  company you are going to have a reply-thread with.
- **`TXT` → DMARC** lives at `_dmarc.<domain>`, so query that name separately if
  you need the policy. Absent DMARC is common and weak evidence on its own.

```
rdap_domain({ domain: "<domain>" })
```

Gives the `registration` event date, the registrar, and the domain status
flags. See `gtm-company-resolution` for the verified output. A domain registered
weeks ago, attached to a list row claiming an established company, is bad data
until proven otherwise.

## A scrub pass that costs nothing

For a list of domains:

1. `dns_lookup_all` each one. Drop rows with zero `MX`.
2. Bucket the survivors by MX provider. Google/Microsoft/Zoho/Proton in one
   bucket, self-hosted in another, single-MX-on-own-hostname in a third.
3. Flag `v=spf1 -all` and any SPF that authorizes nothing.
4. `rdap_domain` on anything that survived but looks thin. Flag registrations
   under ~12 months old.
5. Only now spend verification credits, and only on the survivors.

Report the counts at each stage. "Started with 4,000 domains, 620 had no MX,
190 hard-fail SPF, 3,190 went to verification" is a result a human can act on.
"Cleaned the list" is not.

## Mailbox-level verification: the state of it, honestly

Checked live on 2026-09-01, so you do not waste a call finding out:

- **`check_domain` and `validate_email` (disify) are BYOK now.** The earlier
  `403 — Access denied` was a hostname bug on our side, and fixing it exposed
  what the 403 had been hiding: `429 — Anonymous daily quota exceeded`. Disify
  meters its anonymous tier per IP, our whole fleet shares one egress address,
  and that pool is somebody else's by the time most callers arrive — measured
  2026-09-01, three consecutive calls 429'd while the same URL answered 200
  with full data from a laptop. Pass your own key as `_apiKey` and it goes
  straight through; without one the tools now say so in the error rather than
  looking broken. Pipeworx does not front a key for this pack.
- **`check_email` (IPQualityScore)** returns
  `"You have insufficient credits to make this query."` The shared key is
  exhausted. The tool itself works — pass your own IPQualityScore key as
  `_apiKey` and the call goes straight through.
- **`emailable_verify`** is BYOK-only: `auth_required`, `missing_credentials:
  ["_apiKey"]`, no platform key on that pack. Pass your own Emailable key.

### BYOK is the path, and it is a feature here

Every vendor tool in the catalog accepts `_apiKey` in the arguments. You use
**your own** account with the vendor, at your own contract rate. Nothing is
resold to you and no credits are consumed on a BYOK call.

```
emailable_verify({ email: "...", _apiKey: "<your emailable key>" })
check_email({ email: "...", _apiKey: "<your ipqualityscore key>" })
```

This is the answer to "I already pay for this vendor, why would I pay a markup
to reach it." You do not. The value is the uniform call shape and the composition
with everything else in the catalog, not the resale.

If you hold several verification vendors, run them in your own order and stop at
the first confident verdict — cheapest first, most accurate last. There is no
built-in waterfall tool today; build the ordering in your own code.
