---
name: metabase-api
description: Query the connected Metabase instance (saved questions, dashboards, ad-hoc SQL) using its REST API with static API-key authentication. Covers the x-api-key header, running cards, and native queries.
metadata:
  group: Analytics
  author: agentsbooks
  version: 1.0.0
---

# Metabase API (API-key auth)

The connected Metabase instance is reachable at `$METABASE_URL` and
authenticates with a **static API key** in the **`x-api-key` header** —
**not** a Bearer token, and **not** a session. Both env vars are already set:

```bash
curl -sf -H "x-api-key: $METABASE_API_KEY" "$METABASE_URL/api/user/current"
```

If that returns your user JSON, the connection works. Do not look for a
`$METABASE_ACCESS_TOKEN` — it does not exist.

## Common operations

```bash
H=(-H "x-api-key: $METABASE_API_KEY" -H "Content-Type: application/json")

# List databases / saved questions (cards) / dashboards
curl -sf "${H[@]}" "$METABASE_URL/api/database"
curl -sf "${H[@]}" "$METABASE_URL/api/card"
curl -sf "${H[@]}" "$METABASE_URL/api/dashboard"

# Run a saved question, JSON rows back
curl -sf -X POST "${H[@]}" "$METABASE_URL/api/card/<card-id>/query/json"

# Ad-hoc SQL against database <db-id>
curl -sf -X POST "${H[@]}" "$METABASE_URL/api/dataset" \
  -d '{"type":"native","native":{"query":"SELECT ... LIMIT 100"},"database":<db-id>}'
```

## Notes & gotchas

- `POST /api/dataset` caps results (~2000 rows). For larger extracts use
  `POST /api/card/<id>/query/csv` or add explicit `LIMIT`/pagination in SQL.
- Card parameters go in the body:
  `{"parameters":[{"type":"category","target":["variable",["template-tag","name"]],"value":"..."}]}`.
- A `401`/`403` means the API key is wrong or lacks group permissions — report
  which endpoint failed and the status code; don't retry endlessly.
- Timestamps come back in the instance timezone; state assumptions when you
  report numbers.
