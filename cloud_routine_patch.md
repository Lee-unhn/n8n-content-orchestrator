# Cloud Routine PATCH Template

> Two text snippets to append to your existing Anthropic cloud routine prompts.
> They make the routine write its output to a GitHub Gist that the n8n
> orchestrator can fetch later.

---

## How to apply

1. Open https://claude.ai/code/routines
2. For each routine that should publish content:
   - Paste **PATCH-1** at the **top** of the prompt (rule update)
   - Paste **PATCH-2** at the **bottom** of the prompt (Gist write step)
3. Save the routine
4. Optionally click **Run now** once to verify a Gist gets written

The cloud sandbox has full outbound network access — `curl` to GitHub Gist
API works. Confirmed via spike on 2026-05-01.

---

## PATCH-1 (paste at top of routine prompt)

```
> ⚠️ Rule update
>
> 1. Publish path: this routine no longer relies on a local agent picking up
>    the response. Instead, the routine itself writes to a GitHub Gist (see
>    "Step X: write Gist" appended at the end of this prompt).
> 2. n8n orchestrator pulls the latest Gist daily at the configured cron time
>    and posts to the target platform.
> 3. The routine MUST emit YAML frontmatter at the top of the markdown payload
>    so n8n can parse format / topic / etc.
```

---

## PATCH-2 (paste at bottom of routine prompt)

```
---

# Step X: write the draft to GitHub Gist

After producing the post draft, wrap it in YAML frontmatter and write it to a
secret GitHub Gist. The n8n orchestrator polls Gists and publishes the
latest fb-post-* one.

## Step X.1: build the markdown payload

Run this Bash to write the full markdown (frontmatter + draft) to /tmp/post.md.
Replace `<...>` with values you decided in earlier steps.

```bash
cat > /tmp/post.md << 'EOF'
---
post_date: <YYYY-MM-DD>
formula: <e.g. F-DAILY | F-TOOL | F-RECAP>
format: text
topic: "<one-line topic in your language>"
post_text_chars: <actual char count>
platforms: [fb_page]
tags: [<3-5 tags>]
---

## draft

<full post text>

---

source:
<source name> — <URL>
EOF
```

## Step X.2: POST to GitHub Gist

```bash
DATE=$(date -u +%Y-%m-%d)
PAT='ghp_xxx_REPLACE_WITH_YOUR_GIST_PAT'

python3 -c "
import json
content = open('/tmp/post.md').read()
data = {
  'description': f'fb-post-${DATE}',
  'public': False,
  'files': {f'daily_post_${DATE}.md': {'content': content}}
}
print(json.dumps(data))
" > /tmp/gist_payload.json

GIST_RESP=$(curl -s -X POST https://api.github.com/gists \
  -H "Authorization: token $PAT" \
  -H "Accept: application/vnd.github+json" \
  -d @/tmp/gist_payload.json)

GIST_HTML=$(echo "$GIST_RESP" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('html_url','FAIL'))" 2>&1)
GIST_RAW=$(echo "$GIST_RESP" | python3 -c "import json,sys; d=json.load(sys.stdin); fs=list(d.get('files',{}).values()); print(fs[0].get('raw_url','NO_RAW') if fs else 'NO_RAW')" 2>&1)

echo ""
echo "=== A3 Gist Bridge Result ==="
if [ "$GIST_HTML" != "FAIL" ] && [ "$GIST_HTML" != "" ]; then
  echo "GIST_OK"
  echo "html_url: $GIST_HTML"
  echo "raw_url: $GIST_RAW"
else
  echo "GIST_FAIL"
  echo "Response: $GIST_RESP"
fi
```

## Step X.3: required output

Your final response MUST include:

1. The shortlist + selected item
2. The complete markdown draft (frontmatter + draft body) — even if Gist
   write fails, this is the fallback the user can copy-paste manually
3. The Voice Lock self-checks (if your routine has them)
4. The A3 Bridge Result line: `GIST_OK` + URLs, or `GIST_FAIL` + error
```

---

## Security note

The PAT inside PATCH-2 is stored in your Anthropic cloud routine config.
Treat it as a secret:

- Use a dedicated PAT with **only the `gist` scope** (no `repo`, no `workflow`)
- Mark it "no expiration" only if you accept the trade-off (no auto-rotate)
- Rotate periodically by deleting the PAT on GitHub and replacing it in all
  routines + the n8n workflow's `GIST_PAT` env var
