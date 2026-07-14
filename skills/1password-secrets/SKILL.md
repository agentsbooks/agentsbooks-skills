---
name: 1password-secrets
description: Read secrets (API keys, credentials, config values) from the connected 1Password vault using the preinstalled `op` CLI. Works with both service-account and Connect-server connections — authentication is already done via environment variables.
metadata:
  group: Infrastructure
  author: agentsbooks
  version: 1.0.0
---

# 1Password Secrets (via the `op` CLI)

The 1Password CLI (`op`) is **preinstalled** in this container and **already
authenticated**: the run environment carries either `OP_SERVICE_ACCOUNT_TOKEN`
(service-account connection) or `OP_CONNECT_HOST` + `OP_CONNECT_TOKEN`
(Connect-server connection). You never sign in interactively and you never
need a token value yourself — just run `op` commands.

## When to use this skill

Any time the task needs a credential you don't have in env — an API key
(SERP, OpenAI, …), a login, a webhook URL — and a 1Password connection is
listed under Connected Services. **Do not** try the 1Password REST API with
raw tokens, and do not give up because `$1PASSWORD_ACCESS_TOKEN` is missing —
that variable does not exist. Use `op`.

## Discover what's available

```bash
op vault list --format json                 # vaults you can reach
op item list --vault "<vault>" --format json  # items in a vault (no values)
```

## Read a secret

```bash
# Whole item, fields + values:
op item get "<item-name-or-id>" --vault "<vault>" --format json

# Single field value only (preferred — nothing extra to leak):
op read "op://<vault>/<item>/<field>"

# Typical field names: credential, password, api key, username, notesPlain
```

## Use a secret without printing it

```bash
export SERP_API_KEY="$(op read 'op://Ops/SerpApi/credential')"
curl -s "https://serpapi.com/search?api_key=${SERP_API_KEY}&q=..."
```

## Rules

1. **Never print secret values** into your final output, reports, memory, or
   Slack/posts. Read them into env vars or use them inline in commands.
2. If an item or field is missing, say *which* `op://` reference you tried and
   what `op` returned — don't claim 1Password is unreachable.
3. `op` errors mentioning authorization mean the service account lacks access
   to that vault — report it as a permissions gap for the owner to fix,
   naming the vault.
4. Reads are fast; there is no need to cache secret values in memory or files.
