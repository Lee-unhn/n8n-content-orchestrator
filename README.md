# n8n Content Orchestrator

> Bridge that lets Anthropic cloud routines auto-publish to a Facebook Page,
> using a self-hosted n8n instance and a GitHub Gist as the cloud-to-cloud
> intermediate.
>
> Status: text branch shipped. Image / video branches are out of scope for
> this repo (they live in a separate downstream pipeline).

---

## Why this exists

Cloud-hosted AI routines (Anthropic Code Routines, scheduled remote agents,
etc.) are great at producing content on a schedule — they run with no local
machine attached, do their own web searches, and emit a final response into
a hosted UI.

But they share one fundamental limitation: **they cannot reach back into
your machine, your CMS, or a third-party API to actually publish what they
just wrote.** The output sits in the hosted UI until a human picks it up.

The naive workaround is to run a local agent that reads the cloud routine's
response and publishes it. That works, but it ties your daily publishing to
having a specific machine awake, online, and running a specific tool every
day at 21:00.

This repo replaces that with a different topology:

```
[ Anthropic cloud routine ]
        │
        │  curl POST /gists  (the routine writes its own output to GitHub)
        ▼
[ GitHub Gist (private) ]    ← cloud-to-cloud handoff
        │
        │  HTTP GET /gists   (n8n polls on a cron)
        ▼
[ Self-hosted n8n (Docker) ]
        │
        │  POST /{page_id}/feed
        ▼
[ Facebook Page ]
```

The cloud routine and the Facebook Page never have to know about each other.
GitHub Gist is the asynchronous, durable, always-online middle layer.
n8n is the orchestrator that watches Gist and fans content out to platforms.

---

## What this repo contains

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Spins up n8n locally on port 5678, with timezone + env-var injection |
| `workflow.json` | The 5-node n8n workflow (text branch). Import it into n8n |
| `cloud_routine_patch.md` | Two snippets to paste into your existing cloud routine prompt — they teach the routine to write its draft to Gist |
| `.env.example` | Template for the secrets you need to provide |
| `.gitignore` | Keeps secrets, container data, and editor cruft out of git |

What it does **not** contain:

- Your specific brand voice, formula, or content style. Those belong in your
  cloud routine prompt, not in this orchestrator.
- The image / video publishing branches. Those are a separate downstream
  effort (a video pipeline that calls a media generator like a self-hosted
  short-video tool, plus platform-specific upload modules for YouTube
  Shorts / Instagram Reels).
- Any production secrets — every token in this repo is a placeholder.

---

## Architecture (4 layers)

```
┌─────────────────────────────────────────────────────────────────┐
│ LAYER 1: content generation                                      │
│   Anthropic cloud routine fires on its own cron, runs            │
│   WebSearch / WebFetch, drafts the post text.                    │
│   You append a "Step X: write Gist" tail to its prompt           │
│   (see cloud_routine_patch.md).                                  │
└─────────────────────────────────────────────────────────────────┘
                                │
                                │ A3 cloud-to-cloud bridge
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ LAYER 2: handoff (GitHub Gist)                                   │
│   The cloud routine itself writes a private Gist named           │
│   fb-post-YYYY-MM-DD.md, containing YAML frontmatter + draft.    │
│   No public exposure, no third-party broker, just a Gist.        │
└─────────────────────────────────────────────────────────────────┘
                                │
                                │ HTTP GET /gists (cron-triggered)
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ LAYER 3: orchestration (this repo)                               │
│   n8n in Docker on the user's machine, fires daily at 21:00.    │
│   1. List recent Gists (with the gist-scope PAT)                 │
│   2. Pick latest fb-post-* Gist                                  │
│   3. Fetch the raw markdown                                      │
│   4. Parse YAML frontmatter, extract the draft body              │
│   5. Exchange Facebook system user token → page-scoped token     │
│   6. POST to /{page_id}/feed                                     │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ LAYER 4: publishing                                              │
│   The post lands on the Facebook Page through the Graph API.    │
│   No browser automation, no fragile DOM scraping, no Playwright. │
└─────────────────────────────────────────────────────────────────┘
```

---

## Setup

### Prerequisites

- Docker Desktop (or Docker Engine + Compose v2.20+)
- A GitHub account (for the Gist intermediate)
- A Meta Business Portfolio with a Facebook Page where you have admin access
- An Anthropic Code Routine (this repo assumes you already have one writing
  drafts; the patch in `cloud_routine_patch.md` extends it to also push the
  draft to Gist)

### 1. Generate the secrets

**GitHub PAT (gist scope only):**
1. Go to https://github.com/settings/tokens
2. *Generate new token (classic)* — give it the **`gist`** scope only
3. Pick "no expiration" if you accept the trade-off (no auto-rotate)
4. Copy the token

**Facebook system user token:**
1. In Meta Business Suite, create a Business Portfolio if you don't have one
2. Add your Facebook Page to that portfolio
3. *Settings → Users → System users* → create one with Admin role on the Page
4. Generate a never-expiring access token with permissions:
   `pages_show_list`, `pages_read_engagement`, `pages_manage_posts`
5. Copy the token

Note: this repo's workflow does **not** assume the system user token can
post directly. It exchanges it via `/me/accounts` to a page-scoped token at
runtime — see "Lessons learned" below for why.

### 2. Configure

```bash
git clone https://github.com/<your-username>/n8n-content-orchestrator.git
cd n8n-content-orchestrator
cp .env.example .env
# Edit .env, fill in GIST_PAT, FB_SYSTEM_USER_TOKEN, FB_PAGE_ID
```

### 3. Start n8n

```bash
docker compose up -d
docker compose logs -f n8n   # confirm "n8n ready on ::, port 5678"
```

Open http://localhost:5678, complete the first-run admin setup.

### 4. Import the workflow

In n8n:
1. *Workflows* → *Create workflow* → enter the empty canvas
2. **Ctrl+A → Ctrl+C** the contents of `workflow.json` from your cloned repo
3. Paste into the canvas with **Ctrl+V** — n8n parses the JSON and lays out
   the 5 nodes
4. (Alternative) Some n8n builds expose *Import from File* under the
   `Create workflow` dropdown — pick whichever path your version offers
5. Click **Publish** (formerly "Activate") to arm the schedule

The workflow reads its three secrets from environment variables
(`GIST_PAT`, `FB_SYSTEM_USER_TOKEN`, `FB_PAGE_ID`) which are injected into
the container by `docker-compose.yml` from your `.env` file. No credentials
need to be configured inside n8n's UI.

### 5. Patch your cloud routine

Open `cloud_routine_patch.md` and follow the two-step instructions: paste
PATCH-1 at the top of your routine prompt, PATCH-2 at the bottom. Replace
the placeholder PAT inside PATCH-2 with the same `GIST_PAT` you put in
`.env`. Save the routine.

### 6. End-to-end test

1. In your routine UI, click **Run now** — wait for the routine to finish
2. Check https://gist.github.com/<your-username> — a new private Gist with
   description `fb-post-YYYY-MM-DD` should appear
3. In n8n, click **Execute Workflow** on the imported workflow
4. Watch each node turn green; the final node returns a Facebook post id
5. Open your Facebook Page — the new post is there
6. Done. From now on, the cron-driven path runs unattended every day.

---

## Research process

This is the short version of how this design landed. If you're considering
a similar bridge for your own setup, the failure modes here might save you
some hours.

### Iteration 1: local file watch

The first thought was the obvious one: have the cloud routine emit its
draft, the user copies it into a `next_post.md` file, and n8n watches the
filesystem for changes.

This works, but it doesn't actually decouple the daily publish from the
user being awake at 21:00. The user still has to copy-paste once, every
day. It's not automation, it's a slightly more convenient paste target.

### Iteration 2: pull the routine response via API

The next thought was for n8n to call the cloud-routines API and pull the
last run's response directly. That would be the cleanest path: cloud
routine produces, n8n consumes, no intermediate.

It turned out there is no public "get last run output" endpoint on the
routines API. The hosted UI shows it; the API doesn't expose it. This may
change, but for now it's a dead end.

### Iteration 3: cloud-to-cloud bridge via GitHub Gist

The third design — the one this repo implements — turns the cloud routine
itself into the publisher of the bridge artifact. The routine has a Bash
tool, the Bash tool has `curl`, and GitHub Gist's API accepts a single
HTTP POST with a markdown body. So the routine writes its own output to a
Gist, and n8n on the user's machine fetches it later.

Why GitHub Gist specifically:

- Free, no separate service to provision
- Authenticated read/write with a single PAT
- The cloud sandbox already has outbound network access (verified by spike)
- The data is private (`"public": false`) but accessible via raw URLs
- Gist deletion is trivial, so cleanup is just `DELETE /gists/{id}`

A webhook receiver service would also have worked, but Gist requires zero
infrastructure to stand up.

### A3 spike

Before committing to the design, a one-shot test routine was created that
ran exactly two commands: a `curl https://api.github.com/zen` to verify
outbound, and a `curl POST /gists` with the Gist payload. The Gist showed
up within a second. That confirmed the cloud sandbox could reach
api.github.com and that the PAT-based auth chain worked end-to-end before
any of the production routines were modified.

Lesson: spikes are cheap. Spend the 5 minutes proving the assumption
before you build the architecture on top of it.

---

## Lessons learned

Two non-obvious bugs ate most of the time during the build. Documenting
them here so the next person doesn't burn the same hour.

### Bug 1: n8n HTTP Request node `sendHeaders` is silently ignored after JSON paste

The original workflow used n8n's standard HTTP Request node (typeVersion
4.2) to call `https://api.github.com/gists` with an `Authorization: token`
header. The JSON for the workflow declared:

```json
"sendHeaders": true,
"headerParameters": {
  "parameters": [
    { "name": "Authorization", "value": "token ghp_..." },
    { "name": "Accept", "value": "application/vnd.github+json" }
  ]
}
```

After paste-importing the workflow, the node ran without errors but always
returned the **public** GitHub gist feed instead of the authenticated user's
gists. That's because GitHub's `GET /gists` endpoint, when called without
auth, returns 200 with a global firehose of recent public gists — it
doesn't 401. So the failure is silent.

Investigating from the n8n UI showed `sendHeaders` was visually toggled
**off** in the node panel, even though the JSON had it set to `true`. The
toggle and the underlying flag drift apart on import in some n8n builds.

**Fix**: replace the HTTP Request node with a Code node that calls
`this.helpers.httpRequest()` directly. The Code node has no UI form for
headers — what you write in JS is exactly what gets sent.

The current `workflow.json` uses this pattern for the GitHub Gist call.

### Bug 2: System user permanent tokens cannot directly POST to /{page_id}/feed

The workflow originally tried `POST /{page_id}/feed` with the system user
token in the request body. Facebook returned `403 Forbidden`.

The token was valid (a `GET /me` call returned the system user's id and
name correctly). The system user had Page admin role. The permissions
included `pages_manage_posts`. None of that matters for the publish call.

What's actually required is that the publish call be authenticated with a
**page-scoped** access token, not a user-scoped token, even when the user
*is* an admin of the page.

You exchange one for the other with:

```
GET /me/accounts?access_token=<system_user_token>
```

The response includes a `data` array, one entry per page the user can act
on, each with its own `access_token` field — that's the page-scoped token.
You use that token in the publish call.

This is the same flow that Meta's official `fb_publish.py` example follows,
but it's easy to miss because casual examples often use a single
"page token" that was already exchanged at token-generation time.

The current `workflow.json` does the exchange in a single Code node:
fetch `/me/accounts`, find the matching page id, grab its `access_token`,
then POST to `/{page_id}/feed` with that token in the body.

---

## Maintenance

| Cadence | Task |
|---------|------|
| Daily, 30s | Glance at n8n's *Executions* page — green vs. red |
| Weekly | Skim the executions for any silent failures |
| Monthly | Confirm the GitHub PAT and Facebook system user token are still valid |
| When Meta bumps Graph API version | Update `FB_GRAPH_VERSION` in `.env` and restart |

When something fails:

1. **Gist write failed in the cloud routine** — the routine's final
   response includes `GIST_FAIL` and the raw error. Usually a revoked PAT.
2. **n8n can't find a Gist** — the cloud routine didn't run or didn't
   write. Check the routine's run log in its hosted UI.
3. **n8n found a Gist but parse failed** — the cloud routine emitted bad
   YAML. Open the Gist raw URL, fix the frontmatter, re-execute the
   workflow manually.
4. **Facebook 403** — page-scoped token exchange failed. Re-run with the
   system user token verified directly via `GET /me`.

---

## Out of scope (for this repo)

- **Image branch.** YAML frontmatter has a `format` field for a reason —
  the design anticipates `image` and `video` branches. They are out of
  scope here. If you need them, fork and add a Switch node after the
  parse step that routes by `format`.
- **Multi-platform fan-out.** The current workflow targets a single
  Facebook Page. The architecture is platform-neutral; the publish step
  is a single Code node. To fan out, replace it with a Switch + parallel
  publishers.
- **Content style and brand voice.** Those live in the cloud routine
  prompt, not in the orchestrator.
- **Spam guards / cooldown logic.** Not provided. Add at the publish step
  if your platform's policy requires throttling.

---

## Security

- **Never commit `.env`.** It's in `.gitignore` already.
- **Never paste real tokens into the workflow.json before sharing it.**
  The shipped `workflow.json` reads from `$env`, so it stays clean.
- **Use minimal-scope tokens.** The GitHub PAT only needs `gist`. The
  Facebook system user token only needs `pages_manage_posts`,
  `pages_show_list`, `pages_read_engagement`.
- **Rotate on any suspicion of leakage.** Both PAT and FB token are
  trivially revokable.

---

## License

MIT. See `LICENSE`.
