# n8n Content Orchestrator (v3)

> Cron-based content publishing pipeline. Pulls a daily draft from a GitHub
> Gist, generates or fetches a matching image, then fans out to Facebook
> Page + Instagram + Threads in one shot.
>
> **Status: text + image branches shipped to 3 platforms.** Video branch is
> out of scope.

```
[ Anthropic Cloud Routine — or any LLM ]
        │
        │  curl POST /gists  (writes a private fb-post-YYYY-MM-DD Gist)
        ▼
[ GitHub Gist (private) ]    ← cloud-to-cloud handoff
        │
        │  HTTP GET /gists   (n8n polls on a cron)
        ▼
[ Self-hosted n8n (Docker) ]
        │
        ├─ Resolve Image:
        │    1) scan Drive folder (NEW unprocessed)
        │    2) fallback: Pollinations.ai (Chinese topic auto-translated)
        │
        ├─ Upload to ImgBB (public URL for IG)
        │
        ├─ Publish FB Page (system user → page-scoped → /photos)
        ├─ Publish Instagram (/media → poll → /media_publish)
        └─ Publish Threads (graceful 480-char truncation)
```

---

## Why this design

Cloud-hosted AI routines (Anthropic Code Routines, GitHub Actions cron jobs,
etc.) can produce content on a schedule but **cannot reach back into your
machine, your CMS, or third-party APIs to publish**. The output sits in the
hosted UI until a human picks it up.

The naive workaround — running a local agent that reads the cloud response
and publishes — ties your daily publishing to having one specific machine
awake, online, and running a specific tool every day. Brittle.

This repo replaces that with **GitHub Gist as the asynchronous, durable,
always-online middle layer**. Whatever produces the content (cloud routine
or local LLM or you typing manually) writes a private Gist; n8n reads it on
schedule and fans out to all platforms.

The **content producer and the publishing platforms never have to know
about each other.** Swap any layer independently.

---

## What this repo contains

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Spins up n8n locally on port 5678 with timezone + 16 env-var injection |
| `workflow.json` | The 10-node n8n workflow (text + image branches, three platforms) |
| `cloud_routine_patch.md` | Two snippets to paste into your Anthropic cloud routine prompt to make it write to Gist |
| `.env.example` | Template for all the secrets |
| `.gitignore` | Keeps secrets, container data, editor cruft out of git |

**Not included** (intentionally):
- Brand-specific voice / formula / style — those live in your cloud routine
  prompt, not in this orchestrator
- The video publishing branch — separate phase
- Production secrets — every token in this repo is a placeholder

---

## Quickstart

### Prerequisites
- Docker Desktop (or Docker Engine + Compose v2.20+)
- A GitHub account (for the Gist intermediate)
- A Facebook Page where you have admin access (in a Meta Business Portfolio)
- A way to produce content on a schedule — see "Content sources" below

### 1. Generate the secrets

You need 7 things minimum:

| Secret | Source | Required for |
|--------|--------|--------------|
| **GitHub PAT (`gist` scope)** | https://github.com/settings/tokens (classic) | Always — Gist write/read |
| **Facebook system user token** | business.facebook.com → System users → Generate token | FB Page + IG publishing |
| **Facebook Page ID** | Your Page settings | Always |
| **Instagram Business Account ID** | Linked to the Page; query via Graph API | IG publishing |
| **Threads long-lived token + user_id** | developers.facebook.com → your App → Threads → User token generator | Threads publishing |
| **ImgBB API key** | https://api.imgbb.com → Get API key | IG publishing (public URL host) |
| **Drive API key + folder ID** | Google Cloud Console + a public Drive folder | Optional — image priority source |

See the inline comments in `.env.example` for permission requirements and
exact paths.

### 2. Configure

```bash
git clone https://github.com/<your-username>/n8n-content-orchestrator.git
cd n8n-content-orchestrator
cp .env.example .env
# Edit .env, fill in the secrets you have
```

### 3. Start n8n

```bash
docker compose up -d
docker compose logs -f n8n
# Confirm: "n8n ready on ::, port 5678"
```

Open http://localhost:5678, complete first-run admin setup.

### 4. Import the workflow

1. n8n → *Workflows* → *Create workflow* (empty canvas)
2. **Ctrl+A → Ctrl+C** the contents of `workflow.json`
3. Paste into the canvas with **Ctrl+V** — n8n parses the JSON and lays out
   the 10 nodes
4. Click **Publish** to arm the daily 21:00 schedule

The workflow reads its 16 secrets from environment variables (injected from
`.env` via `docker-compose.yml`). No credentials need to be configured
inside n8n's UI.

### 5. Hook up your content source

Pick one path from "Content sources" below.

### 6. End-to-end test

1. Make sure your content source produces a Gist with description starting
   with `fb-post-`
2. In n8n, click **Execute Workflow** on the imported workflow
3. Watch each node turn green; the Final Log returns three platform results
4. Confirm posts on Facebook + Instagram + Threads
5. Done. From now on, the cron-driven path runs unattended every day.

---

## Content sources

The system is decoupled — anything that writes a `fb-post-YYYY-MM-DD` Gist
works. Pick whichever fits your stack.

### Path A — Anthropic Cloud Routine (the original design)

If you already have a Claude Code subscription, this is the cleanest:

1. Create a routine at https://claude.ai/code/routines
2. Write your content-generation prompt (search news + draft + voice rules)
3. Append `cloud_routine_patch.md`'s PATCH-1 + PATCH-2 to the prompt
4. Set the cron (e.g. daily 20:30)
5. Done — the routine runs in Anthropic's cloud, writes the Gist via curl

### Path B — n8n integrated LLM (recommended for users without Claude Code)

Add a couple of nodes upstream in the n8n workflow itself:

```
Schedule (e.g. 20:30)
  → WebSearch / Perplexity API (HTTP node)
  → LLM API (OpenAI / Gemini / Anthropic; n8n has native nodes for these)
  → Write Gist (HTTP POST to api.github.com/gists)
  → ... existing v3 nodes
```

Cost: a few cents per post (Gemini free tier or GPT-4o-mini ~$0.001/post).

### Path C — GitHub Actions cron (free + cloud)

`.github/workflows/daily-post.yml` running daily, calling Gemini API + curl
Gist. Free tier covers 90 minutes/month easily. No machine needs to stay on.

### Path D — Local cron + Ollama (zero cost, requires GPU)

Windows Task Scheduler / cron calls a local Ollama instance (e.g. `gemma2`),
generates the draft, curls Gist. Fully offline.

### Path E — Manual

You write the Gist by hand at https://gist.github.com daily. Description
must start with `fb-post-`. n8n will pick it up and publish.

---

## Architecture (4 layers)

```
LAYER 1: Content generation
   Anthropic cloud routine / GitHub Action / local LLM / manual
   → outputs markdown with YAML frontmatter + ## draft section
                              ↓
LAYER 2: Handoff (GitHub Gist, private)
   Gist named `fb-post-YYYY-MM-DD`, contains frontmatter + draft body
                              ↓
LAYER 3: Orchestration (this repo, n8n on Docker)
   1. Schedule daily cron
   2. List recent Gists, pick latest fb-post-*
   3. Fetch raw markdown
   4. Parse frontmatter + extract draft
   5. Resolve image:
        - scan a public Drive folder (priority, dedup via staticData)
        - fallback: Pollinations.ai (Chinese topics auto-translated to English)
   6. Upload image to ImgBB → public URL
   7. Publish FB Page (system user → exchange page-scoped token → POST /photos)
   8. Publish Instagram (/media container → poll → /media_publish)
   9. Publish Threads (graceful 480-char truncation at sentence boundary)
  10. Aggregate log
                              ↓
LAYER 4: Publishing
   Facebook Page (Graph API) + Instagram (Graph API) + Threads (Threads API)
```

---

## Lessons learned (debugging notes)

Non-obvious bugs that ate hours during the build. Documenting so the next
person doesn't burn the same time.

### 1. n8n HTTP Request node `sendHeaders` ignored after JSON paste

`sendHeaders: true` in JSON gets visually toggled OFF in the UI on import.
Result: GitHub `/gists` returns 200 with global public Gists (not your
authenticated ones), and the workflow silently picks the wrong content.

**Fix**: use a Code node calling `this.helpers.httpRequest()` directly. JS
auth headers are exactly what gets sent.

### 2. System user permanent token cannot directly POST to /{page_id}/feed

Even with admin role + correct permissions, the token is "user-scoped".

**Fix**: call `GET /me/accounts` first, find your page, grab its
`access_token` field — that's the page-scoped token. Use that for publishing.

### 3. New Instagram / Threads use cases need explicit permission addition

Adding the "Manage Instagram" use case to your Meta App **does not**
auto-grant `instagram_basic` / `instagram_content_publish` to system user
tokens. You must enter the use case → Customize → Add permissions manually,
then regenerate the system user token.

### 4. n8n Code node disallows `require('fs')`

For dedup persistence across runs, **don't reach for the file system**. Use
`$getWorkflowStaticData('global')` — it auto-persists in n8n's database
between runs and survives container restarts (with persistent volume).

### 5. Gemini Imagen requires paid plan for free-tier API keys

Free Gemini API keys can't access Imagen 3/4 or `gemini-2.5-flash-image`
(quota = 0). Switch to **Pollinations.ai** — completely free, no auth, and
its `text` endpoint also handles cheap inline translation if your topic
isn't English.

### 6. Drive API key cannot be merged with Gemini API key

Gemini API has a special service-account requirement that prevents sharing
a single API key across both APIs. Generate a separate Drive-only key.

### 7. Threads token from the Meta App user token generator is already
long-lived (60 days)

You don't need the OAuth `th_exchange_token` step the docs describe. The UI
hands you a 60-day token directly. You only need exchange/refresh later.

### 8. Instagram requires `image_url` in a public domain (no base64)

Even though Facebook's `/photos` endpoint accepts base64, Instagram's
`/media` endpoint only accepts `image_url=https://...`. That's why we route
through ImgBB.

### 9. Threads 500-char limit needs graceful truncation

Naive `slice(0, 500)` cuts mid-sentence. Better: find the last `。` / `. ` /
`！` / `?` / `\n` within the limit and cut there. Pad the limit to 480 to
leave room.

---

## Maintenance

| Cadence | Task |
|---------|------|
| Daily, 30s | Glance at n8n's *Executions* page — green vs. red |
| Weekly | Skim executions for any silent failures |
| Monthly | Confirm GitHub PAT, FB system user token, ImgBB key all valid |
| Within 60 days | Refresh Threads long-lived token (`refresh_access_token` endpoint) |
| When Meta bumps Graph API version | Update `FB_GRAPH_VERSION` in `.env` and restart |

---

## Security

- **Never commit `.env`.** It's in `.gitignore` already.
- **Never paste real tokens into `workflow.json` before sharing.** The
  shipped workflow reads from `$env`, so it stays clean.
- **Use minimal-scope tokens.** GitHub PAT only needs `gist`. FB system user
  token needs only the publishing scopes. Drive API key restricted to Drive
  API only.
- **Rotate on any suspicion of leakage.** Every token is trivially revokable.

---

## Out of scope

- **Image-text alignment beyond translation** — Currently the topic
  auto-translates to drive image generation, but the draft text comes from
  whatever your content source produced (so they share `topic` but aren't
  necessarily on the exact same beat). For deeper alignment, consider
  switching content source to a vision LLM that takes the image as input.
- **Video branch** — Generate short videos from the draft, upload to
  YouTube Shorts / IG Reels / FB Reels. Out of scope for v3.
- **Multi-account fan-out** — Currently targets one FB Page + one IG + one
  Threads. To fan out, replace the publish step with a Switch + parallel
  publishers.
- **Spam guards / cooldown** — None. Add at the publish step if your
  platform's policy requires throttling.

---

## License

MIT. See `LICENSE`.
