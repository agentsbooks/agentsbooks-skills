---
name: google-ads-api
description: Read Google Ads accounts and (when explicitly allowed) change campaigns via the Google Ads REST API using the connected OAuth token + developer token. Use for reporting with GAQL (campaigns, ad groups, keywords, metrics, recommendations) and for guarded spend changes (budgets, pausing, negative keywords). Covers the required headers, the searchStream reporting call, budget-safety rules, and how to wire Google's official read-only MCP server.
metadata:
  group: Advertising
  author: agentsbooks
  version: 1.0.0
---

# Google Ads API

The connected Google Ads account is reached through the **REST API** with two
credentials that are already in the environment:

- `$GOOGLE_ADS_ACCESS_TOKEN` — the user's OAuth token (also in
  `/workspace/connections/google_ads.json`). Sent as `Authorization: Bearer`.
- `$GOOGLE_ADS_DEVELOPER_TOKEN` — the platform developer token, required on
  **every** request as the `developer-token` header.

Plus, when set:

- `$GOOGLE_ADS_CUSTOMER_ID` — the account (customer) id to operate on, digits
  only, no dashes.
- `$GOOGLE_ADS_LOGIN_CUSTOMER_ID` — the manager (MCC) id, only when the account
  is operated through a manager. Sent as the `login-customer-id` header.

Use the current API version **`v24`** (Google ships ~monthly; if a call returns
a version error, check developers.google.com/google-ads/api for the latest and
use that).

```bash
API="https://googleads.googleapis.com/v24"
CID="${GOOGLE_ADS_CUSTOMER_ID}"
H=(-H "Authorization: Bearer $GOOGLE_ADS_ACCESS_TOKEN"
   -H "developer-token: $GOOGLE_ADS_DEVELOPER_TOKEN"
   -H "Content-Type: application/json")
# Add the MCC header only if operating through a manager account:
[ -n "$GOOGLE_ADS_LOGIN_CUSTOMER_ID" ] && H+=(-H "login-customer-id: $GOOGLE_ADS_LOGIN_CUSTOMER_ID")

# Smoke test: list the accounts this token can reach
curl -sf "${H[@]}" "$API/customers:listAccessibleCustomers"
```

## Reporting with GAQL (read-only)

Everything readable comes back through one call — `googleAds:searchStream` — with
a **GAQL** `SELECT`. The response is a JSON **array of batches**, each with a
`results` array; iterate both.

```bash
gaql() {  # gaql "<query>"
  curl -sf -X POST "${H[@]}" "$API/customers/$CID/googleAds:searchStream" \
    -d "$(jq -nc --arg q "$1" '{query:$q}')"
}

# Campaigns + status + budget
gaql "SELECT campaign.id, campaign.name, campaign.status,
      campaign_budget.amount_micros FROM campaign ORDER BY campaign.name"

# 30-day performance per campaign
gaql "SELECT campaign.name, metrics.impressions, metrics.clicks,
      metrics.cost_micros, metrics.conversions, metrics.ctr, metrics.average_cpc
      FROM campaign WHERE segments.date DURING LAST_30_DAYS"

# Ad groups in a campaign
gaql "SELECT ad_group.id, ad_group.name, ad_group.status
      FROM ad_group WHERE campaign.id = <CAMPAIGN_ID>"

# Keyword performance
gaql "SELECT ad_group_criterion.keyword.text, metrics.clicks, metrics.cost_micros
      FROM keyword_view WHERE segments.date DURING LAST_7_DAYS
      ORDER BY metrics.cost_micros DESC LIMIT 50"

# Optimization recommendations
gaql "SELECT recommendation.type, recommendation.campaign, recommendation.dismissed
      FROM recommendation"
```

Money fields are **micros**: `cost_micros`/`amount_micros` ÷ 1,000,000 = the
account currency amount (so `50000000` micros = 50.00). Always convert before
reporting a number, and state the currency.

## Changing campaigns (mutations) — GUARDED

> **Check the mutation policy FIRST.** The run env sets
> `$GOOGLE_ADS_MUTATIONS_ALLOWED`. If it is anything other than exactly `true`,
> you are in **read-only** mode: do **not** call any `:mutate` endpoint — no
> budget, pause/enable, or negative-keyword changes. Report only. When it is
> `true`, you may change campaigns but must never set a daily budget above
> `$GOOGLE_ADS_MAX_DAILY_BUDGET_MICROS` (when that var is set), and must still
> stop for approval if the task has an approval gate.

Writes move real money. Follow these rules and treat every write as
**requiring human sign-off** unless the task has an approval gate that already
covered it:

- **Never raise a budget or un-pause spend without explicit approval.** If the
  task defines an `approval_gate`, STOP and emit the approval request the task
  instructions describe; do not mutate first.
- **Create paused.** Any new campaign/ad group starts `PAUSED` so nothing spends
  before a human confirms.
- **Respect caps.** If the task says a daily-budget ceiling, never exceed it.
- **One change at a time**, and read back the entity afterwards to confirm.

Mutations use per-resource `:mutate` endpoints:

```bash
# Pause a campaign
curl -sf -X POST "${H[@]}" "$API/customers/$CID/campaigns:mutate" -d '{
  "operations":[{"updateMask":"status",
    "update":{"resourceName":"customers/'"$CID"'/campaigns/<CAMPAIGN_ID>","status":"PAUSED"}}]}'

# Change a daily budget (resolve the budget id from the campaign first)
BID=$(gaql "SELECT campaign_budget.id FROM campaign WHERE campaign.id=<CAMPAIGN_ID> LIMIT 1" \
      | jq -r '.[].results[0].campaignBudget.id')
curl -sf -X POST "${H[@]}" "$API/customers/$CID/campaignBudgets:mutate" -d '{
  "operations":[{"updateMask":"amount_micros",
    "update":{"resourceName":"customers/'"$CID"'/campaignBudgets/'"$BID"'","amountMicros":"<MICROS>"}}]}'

# Add a campaign-level negative keyword (spend-reducing, lower risk)
curl -sf -X POST "${H[@]}" "$API/customers/$CID/campaignCriteria:mutate" -d '{
  "operations":[{"create":{"campaign":"customers/'"$CID"'/campaigns/<CAMPAIGN_ID>",
    "negative":true,"keyword":{"text":"<TEXT>","matchType":"BROAD"}}}]}'
```

## Official read-only MCP (optional)

For richer analysis you can wire Google's official **read-only** Google Ads MCP
server (`github.com/googleads/google-ads-mcp`) as a Brain MCP row. It exposes
`search` (GAQL), `list_accessible_customers`, and `get_resource_metadata`, and
reuses the same `$GOOGLE_ADS_DEVELOPER_TOKEN` + OAuth token. It has **no write
tools by design**, so it is safe for reporting agents. Add it as an MCP whose
command is `pipx run --spec git+https://github.com/googleads/google-ads-mcp.git
google-ads-mcp` with the developer token + credentials in its env.

## Gotchas

- `searchStream` returns an **array of batches** — always iterate `.[].results[]`,
  not `.results`.
- A `401` usually means a missing/expired token; a `403 developer-token` error
  means the developer token isn't approved for this account's access level
  (test vs Basic/Standard). Report the status and the exact error `message`;
  don't retry blindly.
- Test-account customers never serve ads (zero impressions/cost) — don't treat
  empty metrics as a bug on a test account.
- This account's data is the client's. Don't paste raw account identifiers or
  spend into any public-facing output; summarize.
