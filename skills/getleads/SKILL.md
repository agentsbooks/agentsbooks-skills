---
name: getleads
description: Enrich people and search the GetLeads contact database over its REST API. Covers the batch endpoints (100 items per request) that replace per-person loops, the async CSV export for large search results, rate limits, credits, and the error codes.
metadata:
  group: Data & Leads
  author: agentsbooks
  version: 1.0.0
---

# GetLeads (person enrichment + contact database)

Base URL `https://app.getleads.io`. Every route except `GET /api/health`
needs your API key, in **either** header (pick one per request):

```bash
curl -sf -H "Authorization: Bearer $KEY" "https://app.getleads.io/api/v1/contacts/health"
# or
curl -sf -H "X-API-Key: $KEY" "https://app.getleads.io/api/v1/contacts/health"
```

Keys look like `glb_live_<keyId>_<secret>`. A missing, malformed or revoked key
returns **401**.

**Finding the key.** It is injected as an environment variable whose name the
task's owner chose — it is *not* guaranteed to be `GETLEADS_API_KEY`. Read the
"Available Secrets" section of your prompt for the exact name, or pick it up
generically rather than guessing:

```python
import os
key = next(v for k, v in os.environ.items()
           if v.startswith("glb_live_"))
```

## Enriching a list: batch, never loop

**This is the one thing to get right.** The enrichment endpoints take **100
people per request** and sustain roughly **10,000 enrichments per minute** at
the default rate limit. The vendor's own guidance: *enrich a list of any size
through them rather than looping one person at a time.*

A 300-person list is **3 requests**, not 300. Looping instead — one call per
person at 20–50 s each — turns a job of seconds into ~95 minutes, which does
not fit a run and cannot be rescued by backgrounding it (see *Run lifetime*
below).

```bash
# 300 LinkedIn URLs → 3 calls
curl -sf -X POST "https://app.getleads.io/api/v1/enrich/from-linkedin" \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"items":[{"linkedin_url":"https://www.linkedin.com/in/example"}, …100 max…]}'
```

Pick the path by what you already know about each person:

| You have | Endpoint | Item key |
|---|---|---|
| Work email | `POST /api/v1/enrich/from-email` | `email` |
| LinkedIn profile URL | `POST /api/v1/enrich/from-linkedin` | `linkedin_url` |
| First + last name + company/domain | `POST /api/v1/enrich/from-person` | see docs |

All three take `{"items": [...]}`. `from-linkedin` also accepts
`limit_per_item` (matches per URL, default 1, max 10).

The response is per-item, and **partial failure is normal** — check each row,
do not assume the whole call failed:

```json
{"ok": true,
 "results": [{"linkedinUrl": "…", "success": true,  "email": "found@company.com", "data": {…}},
             {"linkedinUrl": "…", "success": false, "email": null,               "data": null}],
 "creditsRemaining": 4999}
```

CSV bulk for all three modes: `POST /api/v1/enrich/csv` (multipart `file` +
`mode` + `mapping`, or raw `text/csv`). It is **synchronous and processes the
file inline**, and the gateway times out at 60 s — in practice about **5,000
rows per request**. A larger file returns **504 with no partial result**, so
split into ~5,000-row chunks. The JSON batch endpoints above have no such cap
and are the better default from a script.

## Finding NEW contacts: search, then export

Enrichment answers "tell me about these people." Search answers "find me people
matching these filters." Do not reach for the export to enrich a list you
already hold — that is what the batch endpoints above are for.

- `POST /api/v1/contacts/search` — paged results (`offset`/`limit`,
  `has_more`, `next_offset`). **Every row returned bills 1 credit.**
- `POST /api/v1/contacts/search/count` — free. Size the query before you spend.
- `POST /api/v1/contacts/search/export` — **async**, for large result sets.
  Same filters as search, no `offset`.

The export is the one place polling is correct, and it is cheap:

```
POST /api/v1/contacts/search/export   →  202 {"export_id": "01K…", "job_status": "queued",
                                              "rows_available": 46169, "export_row_cap": 5000}
GET  /api/v1/contacts/search/export/{export_id}
     while job_status is "queued" or "running"  → retry every few seconds
     when  job_status is "completed"            → export_url (presigned, valid 24 h)
```

`max_rows` caps the export (1–50,000). Unlimited plans are additionally subject
to daily (500k) and monthly (6M) export caps; free plans are capped at
`min(creditsRemaining, total_available)`.

## Rate limits

Accounts default to **100 requests per minute across all endpoints**; beyond
that you get **429** — back off and retry. The request rate is separate from
throughput: because batch endpoints take 100 items each, 100 req/min is ~10,000
records/min. If you are anywhere near the limit, you are almost certainly
looping when you should be batching.

## Errors

| Status | Meaning |
|---|---|
| 401 | Missing, malformed, or revoked API key |
| 402 | Out of plan credits, or a daily/monthly export cap — check `GET /api/v1/usage/fair-use` |
| 400 | Invalid body, bad CSV shape, or **batch over the 100-item limit** |
| 429 | Rate limit exceeded — back off |
| 502 | Enrichment provider error or timeout |
| 503 | Contact database unavailable, or funding store empty/not configured |

Error bodies are `{"ok": false, "message": "…", "creditsRemaining": 1234}`
(`creditsRemaining` appears on some 402s only).

## Billing

Two separate balances, and running out of one says nothing about the other:

- **Plan credits** — search, export, enrichment, lookups. 1 credit per item or
  row **where `success` is true**. Counts and filter-values are free. Unlimited
  plans report `creditsRemaining: null` and are governed by fair-usage caps.
- **Prepaid wallet cash** — the LinkedIn scraping products (profile monitoring,
  website visitor identification). These fail with `insufficient_wallet_cash`,
  which means top up the wallet; search and enrichment are unaffected.

## Run lifetime

Your final message ends the run: the container is destroyed and anything you
started is killed. A detached `nohup`/`&` batch does **not** survive, and a
watcher you are still waiting on dies unfinished — the work is lost and the
promised output never lands.

With the batch endpoints this never comes up: a list of any realistic size
finishes in seconds to a couple of minutes, in the foreground. If a job still
will not fit, process what fits, **write the partial result and a note of where
you stopped**, and say so plainly.

## Notes & gotchas

- **Work emails → `/from-email`; LinkedIn URLs → `/from-linkedin`.** Sending
  one to the other is the vendor's most-cited mistake.
- Requests use `snake_case` (`first_name`, `linkedin_url`). Responses add
  `camelCase` convenience keys (`linkedinUrl`, `profileUrl`) **plus** a `data`
  object whose keys mirror the upstream provider (often `snake_case`, e.g.
  `email_address`, `person_linkedin_url`). Read from `data` for provider fields.
- `POST /api/v1/contacts/search/count` is free — always size a query before
  running it, since every returned row costs a credit.
- Health checks: `GET /api/health` (no auth) and
  `GET /api/v1/contacts/health` (auth) — use these to tell "my key is wrong"
  apart from "the service is down" before retrying a batch.
- GetLeads also exposes MCP tools and a CLI (`npm install -g @getleads/cli`).
  The JSON-RPC MCP endpoint is `https://app.getleads.io/api/mcp` — observed
  working in production, though the REST docs describe the MCP surface without
  publishing that URL. Prefer the REST endpoints above from a script: they are
  documented, and the batch shape is explicit. If you do use MCP, send the API
  key header — a headerless request fails as a silent 401.
- Some products (website visitor identification, company followers) are
  UI/MCP-only and are **not** on the `/api/v1` surface.
