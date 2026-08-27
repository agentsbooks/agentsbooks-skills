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
| `/rest/posts` | POST | ✅ | Publish a post — **live and public, once only** |
| `/rest/posts/{URN}` | DELETE | ✅ | Delete your own post — **needs the URN you recorded** |

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

### Step 4 — Deduplicate against your brain

Before commenting, check that you haven't already engaged with this post. Your
engagement log lives in your brain at
`/workspace/brain/kv/linkedin/engagements.json` — an ordinary file, already on
disk if you have run before. Drop already-engaged IDs from your candidate list:

```bash
LOG=/workspace/brain/kv/linkedin/engagements.json
: > /tmp/already_engaged.txt
[ -s "$LOG" ] && jq -r '.. | objects | (.activity_id // .activity_urn // empty) | tostring | gsub("urn:li:activity:"; "")' \
  "$LOG" | grep -E '^[0-9]+$' | sort -u > /tmp/already_engaged.txt
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
returned comment URN to your brain** — it's both proof of engagement and the
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

### Step 7 — Write back to your brain

After each successful comment, rewrite
`/workspace/brain/kv/linkedin/engagements.json` with the full log — the
entries already there plus the new one — under a stable schema. Everything you
write under `/workspace/brain/` is stored automatically when the run ends and
is on disk again next run. Suggested shape:

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

This shape is what makes the skill **cheap on re-runs**: future invocations
dedupe instantly instead of re-discovering. Never put the access token in that
file — a credential in a brain path is refused when the tree is stored and the
run is reported failed.

---

## Publishing a post to your own feed

`POST /rest/posts` publishes **immediately and publicly** to the member's real
feed. There is no draft mode, no preview, and no sandbox on a consumer token.
Every call is visible to their entire network the moment it returns `201`.

### Hard rules

1. **Never publish a test or probe post.** No "Test post", no "please ignore",
   no placeholder copy. If you want to check the token or the API version,
   use `GET /v2/userinfo` — it exercises the same auth and publishes nothing.
2. **Publish exactly once per run.** Compose the final copy first, then make a
   single POST. Never loop the publish call over API versions, retries, or
   phrasings.
3. **A post you cannot prove went out is not a reason to publish again.**
   Losing the URN is a reporting gap, not a delivery failure.

### Step 1 — Pin the API version

`/rest/posts` requires a `LinkedIn-Version` header. Use **`202503`**, which is
active for consumer tokens. If you must confirm a version is live, probe with
a **GET** — never by POSTing to `/rest/posts`:

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  -H "Authorization: Bearer $LINKEDIN_ACCESS_TOKEN" \
  -H "LinkedIn-Version: 202503" \
  "https://api.linkedin.com/v2/userinfo"
```

### Step 2 — Publish once, and capture the URN from the response header

The share URN comes back in the **`x-restli-id` response header**, not the body
(the body is empty on success). Dump the headers with `-D` on the same request
that publishes — this is what removes any need for a second POST:

```bash
PERSON_ID="<your_sub_from_step_1>"
jq -nc \
  --arg author "urn:li:person:${PERSON_ID}" \
  --arg text "$(cat /tmp/post_body.txt)" \
  '{author: $author,
    commentary: $text,
    visibility: "PUBLIC",
    distribution: {feedDistribution: "MAIN_FEED",
                   targetEntities: [],
                   thirdPartyDistributionChannels: []},
    lifecycleState: "PUBLISHED",
    isReshareDisabledByAuthor: false}' > /tmp/post.json

curl -s -X POST -D /tmp/headers.txt -o /tmp/body.txt \
  -H "Authorization: Bearer $LINKEDIN_ACCESS_TOKEN" \
  -H "X-Restli-Protocol-Version: 2.0.0" \
  -H "LinkedIn-Version: 202503" \
  -H "Content-Type: application/json" \
  "https://api.linkedin.com/rest/posts" \
  -d @/tmp/post.json

# 201 = published. The share URN is in the x-restli-id header.
grep -i '^x-restli-id:' /tmp/headers.txt | tr -d '\r' | awk '{print $2}'
```

The post URL is
`https://www.linkedin.com/feed/update/<share-urn>`.

### Step 3 — Interpret the response, then stop

| Response | Meaning | What to do |
|---|---|---|
| `201 Created` | Published | Record the `x-restli-id` URN to your brain. **Stop.** |
| `422 DUPLICATE_POST` | An identical post is **already live** | Your content is published. **Stop.** |
| `401 Unauthorized` | Token expired | Report to the user. Do not retry. |
| `403 ACCESS_DENIED` | Scope missing for this surface | Report. Do not retry on another version. |

**`DUPLICATE_POST` means success, not failure.** LinkedIn is telling you the
content is already on the feed. **Do NOT reword the copy, change punctuation,
or alter a character to get past the duplicate check** — that guard is the only
thing standing between a retry loop and a feed full of near-identical posts.
Treat it as a `201` you already earned, record it, and move on.

If you never captured the URN, say so plainly in your final report
("published, URN not captured") rather than publishing a second time.

### Step 4 — If you published something wrong, delete it by URN

`DELETE /rest/posts/{ENC_URN}` works on a consumer token and returns
**`204 No Content`**. This is the *only* undo available, and it needs the URN
from Step 2 — which is the second reason never to lose it.

```bash
ENC_URN=$(python3 -c "import urllib.parse; print(urllib.parse.quote('urn:li:share:7487402545397780481', safe=''))")
curl -s -o /dev/null -w "%{http_code}\n" -X DELETE \
  -H "Authorization: Bearer $LINKEDIN_ACCESS_TOKEN" \
  -H "LinkedIn-Version: 202503" \
  -H "X-Restli-Protocol-Version: 2.0.0" \
  "https://api.linkedin.com/rest/posts/${ENC_URN}"
```

**Only ever delete a URN you recorded yourself this run.** Never guess or
scan URN values — they are global across all of LinkedIn, and a wrong guess
means deleting an unrelated post of the member's. If you don't have the URN,
report it and let the member remove it from the UI.

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
| `422 DUPLICATE_POST` on `/rest/posts` | The post is **already live** | Treat as success. Record it and stop. Never reword to get around it. |
| `201` on `/rest/posts` but no URN captured | Read the body instead of `x-restli-id` | Report "published, URN not captured". **Never publish again to obtain an ID.** |
| Yahoo returns 0 posts | Search engine guard or bad query | Try Bing; then DuckDuckGo HTML; then `WebFetch`. |
| All search engines empty | Topic too niche, or all results in dedupe list | Broaden keywords; engage fewer posts than requested. |

---

## What this skill explicitly does NOT do

- **Does not** attempt the LinkedIn read API (`/v2/me`, `/rest/posts?q=author`).
  Skip them; they always 403 on consumer tokens.
- **Does not** scrape LinkedIn directly (`https://www.linkedin.com/...`).
  LinkedIn aggressively blocks; web-search discovery is the safe path.
- **Does not** handle DMs or connection requests. Those need additional scopes.
- **Does not** offer any way to publish a draft, a preview, or a test post.
  Everything `POST /rest/posts` sends goes live on the member's real feed.
- **Does not** let you find a post you did not record. Deleting works
  (see below), but only by URN — and `GET /rest/posts` is 403 on a consumer
  token, so there is no way to look one up after the fact. A post whose
  `x-restli-id` you dropped can only be removed by the member, by hand, from
  the LinkedIn UI. Compose carefully, publish once, and record the URN.
