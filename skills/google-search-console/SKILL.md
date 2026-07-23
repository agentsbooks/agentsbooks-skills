---
name: google-search-console
description: Read Google Search Console data via the REST API using the connected OAuth token — search performance (queries, clicks, impressions, CTR, position), verified properties, sitemap status, and URL index inspection. Read-only by design. Covers the searchAnalytics query call, property-URL formats, the data-freshness lag, and error handling.
metadata:
  group: Analytics
  author: agentsbooks
  version: 1.0.0
---

# Google Search Console API

The connected Search Console account is reached through the **REST API** with
one credential that is already in the environment:

- `$GOOGLE_SEARCH_CONSOLE_ACCESS_TOKEN` — the user's OAuth token (also in
  `/workspace/connections/google_search_console.json`). Sent as
  `Authorization: Bearer`.

Plus, when set:

- `$GOOGLE_SEARCH_CONSOLE_SITE_URL` — the default property to query. Domain
  properties look like `sc-domain:example.com`; URL-prefix properties look like
  `https://example.com/`. When unset, discover properties with the sites call
  below and pick the relevant one.

The token is **read-only** (`webmasters.readonly`): you can read performance
data, sitemap status, and index coverage — you cannot submit sitemaps or
change properties. Don't try; the API will 403.

```bash
API="https://www.googleapis.com/webmasters/v3"
H=(-H "Authorization: Bearer $GOOGLE_SEARCH_CONSOLE_ACCESS_TOKEN"
   -H "Content-Type: application/json")

# Smoke test: list the properties this token can read.
# No -f flag anywhere in this skill: on 4xx you NEED the JSON error body
# (and the status code) to report what actually went wrong.
curl -s -w '\nHTTP:%{http_code}\n' "${H[@]}" "$API/sites"
```

## Property URL encoding

The property URL is a **path segment** and must be fully percent-encoded —
including `:` and `/`. Pass it via the environment (never splice it into
Python source):

```bash
SITE="${GOOGLE_SEARCH_CONSOLE_SITE_URL:-sc-domain:example.com}"
SITE_ENC=$(SITE="$SITE" python3 -c "import urllib.parse,os;print(urllib.parse.quote(os.environ['SITE'], safe=''))")
```

## Search performance (the main call)

Everything about queries/pages/CTR/position comes from one endpoint:
`searchAnalytics/query`. **Data lags ~2 days** — never ask for a range ending
today; end 2 days ago.

```bash
END=$(date -d '2 days ago' +%F 2>/dev/null || date -v-2d +%F)
START=$(date -d '30 days ago' +%F 2>/dev/null || date -v-30d +%F)

# Top queries, last 28 days
curl -s -w '\nHTTP:%{http_code}\n' -X POST "${H[@]}" "$API/sites/$SITE_ENC/searchAnalytics/query" -d "{
  \"startDate\": \"$START\", \"endDate\": \"$END\",
  \"dimensions\": [\"query\"], \"rowLimit\": 100
}"

# Top pages
#   \"dimensions\": [\"page\"]
# Query × page (find which page ranks for which query)
#   \"dimensions\": [\"query\", \"page\"]
# Daily trend
#   \"dimensions\": [\"date\"]
```

Each row: `{"keys": [...], "clicks": n, "impressions": n, "ctr": 0.0..1.0,
"position": avg}`. `ctr` is a fraction (multiply by 100 for %). Rows are
sorted by clicks descending.

**Emerging-opportunity pattern**: high impressions + low CTR + position 11–30
= content that almost ranks. Filter rows client-side after fetching with
`"rowLimit": 1000`.

## Sitemaps (read)

```bash
curl -s -w '\nHTTP:%{http_code}\n' "${H[@]}" "$API/sites/$SITE_ENC/sitemaps"
```

## URL inspection (index status of one page)

Different host — the newer Search Console API:

```bash
curl -s -w '\nHTTP:%{http_code}\n' -X POST "${H[@]}" \
  "https://searchconsole.googleapis.com/v1/urlInspection/index:inspect" \
  -d "{\"inspectionUrl\": \"https://example.com/page\", \"siteUrl\": \"$SITE\"}"
```

## Errors — and the honesty rule

- **403** — the connected Google account isn't a user/owner of that property,
  or the property URL format is wrong (`sc-domain:` vs URL-prefix). Re-check
  with the sites call.
- **404** — property not found for this account; use the exact `siteUrl`
  string from the sites call.
- **401** — token expired; report it so the owner can reconnect.

If Search Console data is unavailable for ANY reason, **say so in your output
and stop the analysis step** — report the exact error and which credential/
scope is missing. Never substitute estimated, synthesized, or made-up numbers
for real Search Console data; downstream steps (posts, reports, PRs) must not
present invented metrics as measured ones.
