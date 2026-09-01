---
name: gtm-headless
description: Call Pipeworx tools without an MCP server — from the terminal via the `pipeworx` CLI, or over plain HTTP JSON-RPC from any language. Use when the user works CLI-only, wants results piped into jq or a script, is scripting a cron or a data pipeline, or has asked how to avoid loading an MCP server into context.
---

# Headless: calling tools without MCP

MCP is one transport, not the product. If the caller is a coding harness, a
cron, or a script, the MCP server is overhead — a process to start, a tool list
to load into context, and a class of auth failure that has nothing to do with
the data. Two other paths exist and neither needs a server.

Use this skill when someone says any of: "no MCP", "CLI only", "just give me
curl", "I want to pipe it into jq", "I'm running this on a schedule".

## Path 1 — the CLI

```
npx pipeworx@latest call <tool_name> --key=value --key=value
```

The contract, which is what makes it scriptable:

- **stdout is the tool's JSON payload and nothing else, ever.** No "Calling…",
  no "Result:". `| jq` works on line one.
- **stderr carries every human-facing word**, including errors.
- **exit 0** = the tool returned data. **exit 1** = the tool errored, the
  arguments were wrong, or the call failed. So `|| echo failed` means something.

Argument coercion: `--limit=3` arrives as the number `3`, not `"3"`, because
several packs type-check and reject the string. When a value must stay a string
— a zip code like `01234`, an ID that looks numeric — use `--json '{"zip":"01234"}"'`
instead; JSON is passed through untouched.

Other flags: `--api-key`, `--pack` (skip pack resolution), `--timeout`,
`--json`, `--help`. Everything else in `--k=v` form is a **tool** argument.

Do not reach for `pipeworx test` in a pipeline. It prints prose to stdout
alongside the data and sends no credentials — it is a human sanity check.
`call` is the machine surface.

## Path 2 — plain HTTP JSON-RPC

For any language with no Node.js in the picture. One endpoint, one POST.

```
POST https://gateway.pipeworx.io/mcp
Content-Type: application/json

{ "jsonrpc": "2.0", "id": 1, "method": "tools/call",
  "params": { "name": "<tool_name>", "arguments": { ... } } }
```

`method: "tools/list"` with `params: {}` enumerates every live tool and its
input schema, including per-tool `examples`. Read the examples before guessing
argument names — several tools reject a plausible-looking key rather than a
wrong value, and a few silently ignore one, which is worse.

### Reading the response

The payload you want is `result.structuredContent` — **top level, not nested
under an `answer` key.** If `structuredContent` is absent, fall back to
`result.content`. Check `result.isError` before parsing: on a tool error the
prose verdict arrives inside `result.content[].text`, not in a JSON-RPC `error`
object. A JSON-RPC `error` at the top level means the envelope was wrong
(bad method, malformed params), which is a different bug from the tool failing.

### Auth

Send one of these; they resolve in this order:

| Header | Tier | Daily limit |
|---|---|---|
| `Authorization: Bearer <token>` | OAuth (free GitHub sign-in) or paid | 200 / unlimited |
| `X-API-Key: <key>` | BYO | 200 |
| *(none)* | anonymous, per IP | 50 |

Anonymous works for a first call with no signup at all. Vendor-backed tools take
your own vendor key inline as an `_apiKey` **argument** — see
`gtm-email-hygiene`.

## Verified invocation

Sent as a plain JSON-RPC POST to `https://gateway.pipeworx.io/mcp` on
2026-09-01:

```json
{ "jsonrpc": "2.0", "id": 1, "method": "tools/call",
  "params": { "name": "h1b_top_sponsors",
              "arguments": { "job_title": "data engineer", "limit": 3 } } }
```

`result.structuredContent`:

```json
{
  "job_title": "data engineer",
  "year": 2025,
  "city": null,
  "total_filings": 4257,
  "top_sponsors": [
    { "employer": "META PLATFORMS INC",           "lca_filings": 211, "median_base_salary": 197995 },
    { "employer": "IBM CORPORATION",              "lca_filings": 63,  "median_base_salary": 121514 },
    { "employer": "COMPUNNEL SOFTWARE GROUP INC", "lca_filings": 49,  "median_base_salary": 124000 }
  ],
  "note": "Employers ranked by certified H-1B LCA filings for this role — a sourcing/target-account signal. Source: DOL LCA disclosures."
}
```

The equivalent CLI form:

```
npx pipeworx@latest call h1b_top_sponsors --job_title="data engineer" --limit=3 | jq '.top_sponsors[].employer'
```

Note `total_filings: 4257` sitting next to three returned rows. That is the
denominator, and it belongs in whatever you build on top — see
`gtm-derived-signals`.

## Errors are written to be acted on

A missing argument comes back naming the argument, the full required list, and a
worked example:

```json
{ "error": "invalid_arguments",
  "message": "h1b_top_sponsors received an invalid argument: job_title is required but was not provided. Required: job_title.",
  "missing": ["job_title"], "required": ["job_title"],
  "retry_hint": "Retry the same tool with arguments matching one of the example shapes.",
  "examples": [ { "job_title": "data engineer", "city": "AUSTIN", "year": 2024 },
                { "job_title": "nurse", "limit": 20 } ] }
```

An auth-gated tool says which credential is missing and that nothing else was
checked yet, so you do not chase a second bug that does not exist:

```json
{ "error": "auth_required",
  "message": "emailable_verify needs your own API key: _apiKey required. Nothing else about the call was checked — supply the key first.",
  "missing_credentials": ["_apiKey"],
  "note": "Other required arguments (email) were not validated yet." }
```

Both shapes are stable enough to branch on in a retry loop: read `error`, then
`missing` or `missing_credentials`, then `examples`.
