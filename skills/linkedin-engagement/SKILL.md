---
name: linkedin-engagement
description: Find and engage with LinkedIn posts (comment, like) using only the consumer-grade `w_member_social` OAuth scope. Works around LinkedIn's restricted `r_member_social` read scope by discovering post URLs via public web search.
metadata:
  group: Social platforms
  author: agentsbooks
  version: 1.0.0
---

# LinkedIn Engagement (consumer-scope safe)

This skill lets an agent **find, comment on, and like LinkedIn posts** using
only the OAuth scopes a typical consumer integration is granted:

- `email`, `openid`, `profile` — identity (OIDC)
- `w_member_social` — write comments and likes on behalf of the member

It explicitly **does not require** `r_member_social`, which LinkedIn restricts
to approved partners and which most consumer integrations cannot get. The
trade-off: the agent cannot read its own feed directly via the LinkedIn API.
Instead, this skill discovers post URLs through **public web search engines**
and engages by URN.

---

## When to use this skill

Use this skill any time you have a LinkedIn token mounted at
`/workspace/connections/linkedin.json` (or a `$LINKEDIN_ACCESS_TOKEN`
env var) and the task asks you to **comment on, like, or engage with**
LinkedIn posts. Do **not** waste turns probing read-only endpoints —
they will 403 with `w_member_social`. Jump straight to the workflow below.

---

## What works and what doesn't (with `w_member_social` only)

| Endpoint | Method | Works? | Notes |
|---|---|---|---|
| `/v2/userinfo` | GET | ✅ | OIDC; returns `sub` = your person ID |
| `/v2/me` | GET | ❌ 403 | Needs `r_liteprofile` or `r_member_social` |
| `/rest/posts?q=author` | GET | ❌ 403 | Needs `r_member_social` (partner-gated) |
| `/v2/socialActions/{URN}/comments` | POST | ✅ | Write a comment |
| `/v2/socialActions/{URN}/comments` | GET | ⚠️ | Often 403; treat as unavailable |
| `/v2/socialActions/{URN}/likes` | POST | ✅ | Add a like |
| `/rest/posts` | POST | ✅ | Create a new post |

**Anti-pattern:** trying `/v2/me`, then `/rest/posts?q=author`, then giving up
when both 403. That wastes ~$0.005 per try and produces no engagement.
Use the workflow below instead.

---

## Workflow

### Step 1 — Get your person URN

```bash
curl -s -H "Authorization: Bearer $LINKEDIN_ACCESS_TOKEN" \
  "https://api.linkedin.com/v2/userinfo" | jq -r .sub
```

The `sub` field is your LinkedIn person ID. Your author URN is
`urn:li:person:<sub>` — you'll need it on every comment/like POST.

### Step 2 — Discover post URLs via web search

LinkedIn-indexed posts are reachable via `site:linkedin.com/posts` queries
on public search engines. **Yahoo's HTML search** tends to return the most
parsable results; Bing and DuckDuckGo are good fallbacks. Avoid Google —
it's the most hostile to scraping.

Pick keywords matching the task topic, then:

```bash
# Yahoo (most reliable for site:linkedin.com)
curl -sG "https://search.yahoo.com/search" \
  --data-urlencode "p=site:linkedin.com/posts <YOUR KEYWORDS> 2026" \
  -H "User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36" \
  -o /tmp/results.html

# Extract post URLs (these look like .../posts/<slug>-activity-<digits>-<random>)
grep -oE 'linkedin\.com/posts/[a-zA-Z0-9_%-]+activity-[0-9]+[a-zA-Z0-9_-]*' /tmp/results.html \
  | sort -u
```

If Yahoo gives you nothing useful for the topic, retry with **Bing**:

```bash
curl -sL "https://www.bing.com/search?q=site%3Alinkedin.com%2Fposts+<URL_ENCODED_KEYWORDS>" \
  -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36" \
  -o /tmp/bing.html

grep -oE 'linkedin\.com/posts/[a-zA-Z0-9_%.-]+activity-[0-9]+[a-zA-Z0-9_-]*' /tmp/bing.html | sort -u
```

If still nothing, try **DuckDuckGo's HTML endpoint** (`html.duckduckgo.com`,
NOT `duckduckgo.com/html/` which returns JS). And as a last resort, the
`WebFetch` tool against any of these search-result URLs with a prompt like:
*"Extract every URL containing `linkedin.com/posts/...activity-...` and
return them, one per line."*

### Step 3 — Extract activity IDs and build URNs

From any `linkedin.com/posts/...-activity-<NNNNNNNNNNNNNNNNNNN>-<random>`
URL, the long digit string is the activity ID. Build the URN:

```bash
ACTIVITY_ID="7460192558099574786"   # extracted from URL
URN="urn:li:activity:${ACTIVITY_ID}"
ENC_URN=$(python3 -c "import urllib.parse; print(urllib.parse.quote('$URN', safe=''))")
```

The URL-encoded form (`urn%3Ali%3Aactivity%3A7460...`) is what the
`socialActions` endpoint requires in its path.

### Step 4 — Deduplicate against memory

Before commenting, check that you haven't already engaged with this post.
Memory at `/workspace/memory/input.json` should track prior `activity_id`s
under a stable schema. Drop already-engaged IDs from your candidate list:

```bash
# Build a sorted list of activity IDs you've already engaged with
jq -r '.. | objects | (.activity_id // .activity_urn // empty) | tostring | gsub("urn:li:activity:"; "")' \
  /workspace/memory/input.json | grep -E '^[0-9]+$' | sort -u > /tmp/already_engaged.txt
```

### Step 5 — Post a comment

```bash
PERSON_ID="<your_sub_from_step_1>"
curl -s -X POST \
  -H "Authorization: Bearer $LINKEDIN_ACCESS_TOKEN" \
  -H "X-Restli-Protocol-Version: 2.0.0" \
  -H "Content-Type: application/json" \
  "https://api.linkedin.com/v2/socialActions/${ENC_URN}/comments" \
  -d "$(jq -nc \
        --arg actor "urn:li:person:${PERSON_ID}" \
        --arg msg   "Your short, engaging comment goes here." \
        '{actor: $actor, message: {text: $msg}}')"
```

A successful response is `201 Created` with a JSON body containing
`"id": "<commentUrn>"` and `"$URN": "urn:li:comment:(...)"`. **Save the
returned comment URN to memory** — it's both proof of engagement and the
dedupe key for next run.

### Step 6 — Add a like

```bash
curl -s -X POST \
  -H "Authorization: Bearer $LINKEDIN_ACCESS_TOKEN" \
  -H "X-Restli-Protocol-Version: 2.0.0" \
  -H "Content-Type: application/json" \
  "https://api.linkedin.com/v2/socialActions/${ENC_URN}/likes" \
  -d "$(jq -nc --arg actor "urn:li:person:${PERSON_ID}" \
        '{actor: $actor}')"
```

Likes can sometimes return `409 Conflict` if you already liked the post —
that's fine, treat as success.

### Step 7 — Write back to memory

After each successful comment, append to `/workspace/memory/output.json`
under a stable schema. Suggested shape:

```json
{
  "schema_version": 2,
  "person_urn": "urn:li:person:<id>",
  "engagements": [
    {
      "activity_id": "7460192558099574786",
      "post_url": "https://www.linkedin.com/posts/<slug>",
      "comment_id": "urn:li:comment:(...)",
      "comment_text": "...",
      "engaged_at": "2026-05-21T09:41:42Z",
      "discovery": "yahoo:site:linkedin.com/posts AI agents 2026"
    }
  ],
  "discovery_methods_that_worked": [
    "yahoo search site:linkedin.com/posts + topic keywords",
    "bing search site:linkedin.com/posts + topic keywords"
  ]
}
```

This memory shape is what makes the skill **cheap on re-runs**: future
invocations dedupe instantly instead of re-discovering.

---

## Comment quality rules

- Keep comments short (≤30 words).
- Be specific to the post's content — generic "great post!" gets flagged.
- Match the post's language (English ↔ English, Hebrew ↔ Hebrew, etc.).
- Don't link out unless explicitly asked.
- Don't post identical comments across multiple posts in the same run.

---

## Failure modes and recovery

| Symptom | Cause | Recovery |
|---|---|---|
| `403 ACCESS_DENIED` on `/v2/me` | Missing read scope | Expected. Skip this endpoint entirely. |
| `403 partnerApiPostsExternal` on `/rest/posts?q=author` | Missing `r_member_social` | Expected. Use web-search discovery instead. |
| `401 Unauthorized` on any POST | Token expired | Report to user; cannot self-recover without re-auth. |
| `429 Too Many Requests` | Engaging too fast | Sleep 30s and continue; cap at ~10 engagements per run. |
| Yahoo returns 0 posts | Search engine guard or bad query | Try Bing; then DuckDuckGo HTML; then `WebFetch`. |
| All search engines empty | Topic too niche, or all results in dedupe list | Broaden keywords; engage fewer posts than requested. |

---

## What this skill explicitly does NOT do

- **Does not** attempt the LinkedIn read API (`/v2/me`, `/rest/posts?q=author`).
  Skip them; they always 403 on consumer tokens.
- **Does not** scrape LinkedIn directly (`https://www.linkedin.com/...`).
  LinkedIn aggressively blocks; web-search discovery is the safe path.
- **Does not** handle DMs, connection requests, or content beyond comments
  and likes. Those need additional scopes.
