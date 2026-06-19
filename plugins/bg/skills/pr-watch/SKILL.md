---
name: pr-watch
description: Poll GitHub for new pull requests from a configured set of authors, review each new one, save an HTML report, and notify. Run "/bg:pr-watch" to start - it self-schedules on an interval. Pass a directory to use as the reviews folder, e.g. "/bg:pr-watch ~/pr-reviews".
user_invocable: true
model: sonnet
allowed-tools:
  - Bash
  - Read
  - Write
  - Agent
  - ScheduleWakeup
  - AskUserQuestion
  - Skill
  - mcp__plugin_slack_slack__slack_search_users
  - mcp__plugin_slack_slack__slack_send_message
---

# bg:pr-watch

Polls GitHub for open PRs matching a configured search, finds the ones that appeared since the last poll, spawns a review subagent for each, saves an HTML report next to the config, sends a notification, and reschedules itself.

Everything for one watcher lives in a single **reviews directory**: `config.json`, `.state.json`, and the dated HTML reports. Move or back up that one folder and you have the whole setup.

Be concise in your turn output. The work is mostly bash plus subagents; do not narrate every step.

## Step 1 - Resolve the reviews directory and load config

The reviews directory is resolved in this order:
1. The first argument to the skill, if given (e.g. `/bg:pr-watch ~/dd/tasks/pr-reviews`).
2. The `$PR_WATCH_DIR` environment variable, if set.
3. Default: `~/pr-reviews`.

Substitute the resolved path for `REVIEWS_DIR` in every command below. Run this first:

```bash
REVIEWS_DIR="<resolved path, with ~ expanded to $HOME>"
mkdir -p "$REVIEWS_DIR"
CONFIG="$REVIEWS_DIR/config.json"
if [ -f "$CONFIG" ]; then echo "CONFIG_OK"; cat "$CONFIG"; else echo "CONFIG_MISSING"; fi
```

- If it prints `CONFIG_OK`, parse the JSON that follows and continue to Step 2.
- If it prints `CONFIG_MISSING`, run **first-run setup** (below), then continue.

### First-run setup

Use `AskUserQuestion` to gather the essentials, then write `config.json`. Ask for:
1. The GitHub search query (the part after `q=`). Offer the user's last-known query if you have one, otherwise explain the format: `org:ORG is:pr is:open -is:draft author:USER1 author:USER2 ...`.
2. Notification method: `slack`, `webhook`, or `none`.

Write the config using the schema in the Appendix, filling unanswered fields with the documented defaults. Then continue to Step 2.

## Step 2 - Fetch current PRs and compute the new ones

```bash
REVIEWS_DIR="<resolved path>"
CONFIG="$REVIEWS_DIR/config.json"
STATE="$REVIEWS_DIR/.state.json"
[ -f "$STATE" ] || echo '{"seen_prs":[]}' > "$STATE"

QUERY=$(jq -r '.search.query' "$CONFIG")
PER_PAGE=$(jq -r '.search.per_page // 100' "$CONFIG")

CURRENT=$(gh api -X GET search/issues \
  -f q="$QUERY" -f sort=created -f order=desc -F per_page="$PER_PAGE" \
  --jq '[.items[] | {url:.html_url, title:.title, author:.user.login, repo:(.repository_url|split("/")|last), number:.number, created_at:.created_at}]')

echo "$CURRENT" > "$REVIEWS_DIR/.current.json"
SEEN=$(jq -c '.seen_prs // []' "$STATE")
echo "=== NEW PRS ==="
echo "$CURRENT" | jq --argjson seen "$SEEN" '[.[] | select(.url as $u | ($seen | index($u)) | not)]'
echo "=== COUNT ==="
echo "$CURRENT" | jq --argjson seen "$SEEN" '[.[] | select(.url as $u | ($seen | index($u)) | not)] | length'
```

- If the count is `0`, there is nothing new. Skip to Step 5.
- Otherwise, the `NEW PRS` JSON array is the work list for Step 3.

## Step 3 - Review each new PR (one subagent per PR, in parallel)

For each new PR, spawn a subagent with the **Agent** tool. Send all of them in a single message so they run in parallel.

- `subagent_type`: `claude`
- `model`: read `review.model` from config (default `claude-opus-4-8`, i.e. `opus`)
- `description`: `Review PR <number>`

Build each subagent's prompt from the template below, substituting the PR fields and the relevant config values (`repo_base_path`, `review.skills`, `issue_tracker.*`, `notify.*`, and `REVIEWS_DIR`).

> You are reviewing a single GitHub pull request end to end. Work autonomously and thoroughly; this is a high-effort review.
>
> PR: `<url>` | Title: `<title>` | Author: `<author>` | Repo: `<repo>` | Number: `<number>`
> Reviews directory: `<REVIEWS_DIR>`
> Repo base path: `<repo_base_path>`
> Review skills setting: `<review.skills>`
> Issue tracker: pattern `<issue_tracker.pattern>`, base URL `<issue_tracker.base_url>`
> Notify: type `<notify.type>`, slack recipient `<notify.slack_recipient>`, webhook `<notify.webhook_url>`
>
> 1. Gather context: `gh pr view <url> --json title,body,author,additions,deletions,changedFiles,files,headRepository,headRepositoryOwner,headRefName,baseRefName`. Also fetch the diff with `gh pr diff <url>`.
>
> 2. Check out the PR without disturbing any existing work, using a git worktree:
>    - `REPO_PATH="<repo_base_path>/<repo>"` (expand `~`). If it does not exist, clone it: `gh repo clone <headRepositoryOwner>/<repo> "$REPO_PATH"`.
>    - `git -C "$REPO_PATH" fetch origin pull/<number>/head:pr-<number>` then `WT=$(mktemp -d /tmp/pr-watch-<number>-XXXX)` and `git -C "$REPO_PATH" worktree add "$WT" pr-<number>`.
>    - If checkout fails for any reason, fall back to reviewing the diff from `gh pr diff` directly and note this in the report.
>
> 3. Decide which review approach to use:
>    - If the review skills setting is an explicit list, use the listed skill(s).
>    - If it is `"auto"`: list directories under `~/.claude/plugins/marketplaces/*/plugins/*/skills/` and `~/.claude/skills/`. If the changed files are more than 50% Go (`.go`) and a Go review skill exists (e.g. `go-code-reviewer:go-review`), use it. Otherwise, if a generic PR review skill exists (e.g. `pr-review-toolkit:review-pr`), use it.
>    - If the setting is an empty list, or no suitable skill is installed, perform the **built-in review** yourself (below).
>    - Run the chosen skill from inside the worktree directory via the Skill tool. If invoking the skill fails, fall back to the built-in review. Either way you must end up with concrete findings.
>
>    Built-in review: read the changed files in full (not just the diff hunks) and assess: correctness (logic errors, off-by-ones, unchecked errors, nil/undefined), security (injection, auth bypass, secret leakage, unsafe deserialization, SSRF), test coverage (new code paths without tests), error handling (silent failures, swallowed errors), style and naming consistency with surrounding code, and performance in hot paths. Give every finding a `file:line` reference and a concrete suggested fix.
>
> 4. Extract issue-tracker tickets: match the pattern against the title and body. When building a link, normalize the separator to a hyphen (e.g. `SEC 33114` becomes `SEC-33114`) and append to the base URL. If no base URL is configured, list the IDs as plain text.
>
> 5. Write the HTML report (use the exact structure in the skill's Appendix) to:
>    `<REVIEWS_DIR>/<YYYY-MM-DD>_<repo>_<author>_<slug>.html`
>    where `<slug>` is the first ~5 words of the title, lowercased and kebab-cased (strip the ticket prefix and punctuation). Use today's date.
>
>    The report is a working tool for a programmer reviewing the PR, so prioritise functional content over decoration. Each finding card MUST include: the severity, a short title, a clickable `file:line` link to the exact line on GitHub (`https://github.com/<headRepositoryOwner>/<repo>/blob/<headRefName>/<path>#L<line>`), a real code excerpt of the offending lines pulled from the worktree (not a paraphrase), a clear explanation of the problem and its impact, and a concrete suggested fix as a code block. Also include a Context section, a Top Actions list (each item linking to its finding), a Changed files list (each path linking to the file in the PR with its +/- counts), and a per-file collapsible diff section built from `gh pr diff`. HTML-escape every code excerpt and diff (`&` -> `&amp;`, `<` -> `&lt;`, `>` -> `&gt;`); colour added/removed diff lines. For a very large file, include a truncated diff with a note and the GitHub link rather than omitting it silently.
>
>    Writing conventions for prose: clear and direct, no emojis in prose (the template's UI emojis and diff markers are fine), no em-dashes, do not assume the PR is production-ready, hyperlink any tickets.
>
> 6. Clean up the worktree: `git -C "$REPO_PATH" worktree remove "$WT" --force` and `git -C "$REPO_PATH" branch -D pr-<number>` (ignore errors).
>
> 7. Notify, based on the notify type. First build the review link as a `file://` URL from the absolute report path (e.g. `file:///Users/you/pr-reviews/2026-06-19_repo_author_slug.html`) so it opens in the browser.
>    - `slack`: find the recipient with `mcp__plugin_slack_slack__slack_search_users` (query = the configured recipient name), then `mcp__plugin_slack_slack__slack_send_message` to their DM, using the markdown layout in the skill's notification template (title, subtitle, summary, findings line, and clickable Review and PR links).
>    - `webhook`: `curl -fsS -X POST -H 'Content-Type: application/json' -d '<json>' "<webhook_url>"` with fields `{title, author, repo, summary, pr_url, report_path, file_url}`.
>    - `none`: skip.
>
> 8. Return a one-line status: the PR URL, the saved report filename, and whether the notification was sent. If you could not save a report, say so and why.

## Step 4 - Update state

After all subagents return, update `seen_prs` so reviewed PRs are not re-reviewed, while pruning PRs that are no longer open (keeps state bounded). Treat a PR as processed only if its subagent reported a saved report.

```bash
REVIEWS_DIR="<resolved path>"
STATE="$REVIEWS_DIR/.state.json"
CURRENT_URLS=$(jq -c '[.[].url]' "$REVIEWS_DIR/.current.json")
# PROCESSED = JSON array of PR URLs whose subagent saved a report this run.
PROCESSED='<JSON array of processed PR URLs>'
OLD_SEEN=$(jq -c '.seen_prs // []' "$STATE")
NEW_SEEN=$(jq -nc --argjson old "$OLD_SEEN" --argjson done "$PROCESSED" --argjson cur "$CURRENT_URLS" \
  '(($old + $done) | unique) as $union | [ $union[] | select(. as $u | $cur | index($u)) ]')
jq -n --argjson seen "$NEW_SEEN" '{seen_prs:$seen}' > "$STATE"
echo "State updated: $(echo "$NEW_SEEN" | jq 'length') seen PRs."
```

A new PR whose review failed is intentionally left out of `seen_prs`, so it is retried on the next poll.

## Step 5 - Reschedule

Reschedule the next poll. Read `interval_seconds` from config (default `600`). Pass the resolved reviews directory back as the argument so the location persists across polls.

Call `ScheduleWakeup` with `delaySeconds` = `interval_seconds`, `prompt` = `/bg:pr-watch <REVIEWS_DIR>`, and a short `reason` such as "polling team PRs every 10m".

Then end the turn with a one-line summary: how many new PRs were reviewed and when the next poll fires.

## Appendix

### config.json schema

```json
{
  "search": {
    "type": "github_rest",
    "query": "org:ORG is:pr is:open -is:draft author:USER1 author:USER2",
    "per_page": 100
  },
  "interval_seconds": 600,
  "repo_base_path": "~/src",
  "review": {
    "model": "claude-opus-4-8",
    "effort": "high",
    "skills": "auto"
  },
  "issue_tracker": {
    "pattern": "[A-Z]+-[0-9]+",
    "base_url": "https://your-org.atlassian.net/browse/"
  },
  "notify": {
    "type": "slack",
    "slack_recipient": "Your Name",
    "webhook_url": null
  }
}
```

Defaults when a field is omitted: `search.per_page` 100, `interval_seconds` 600, `review.model` `claude-opus-4-8`, `review.effort` `high`, `review.skills` `"auto"`, `notify.type` `none`. Set `issue_tracker.base_url` to `null` to show ticket IDs without links. `review.skills` accepts `"auto"`, an empty list `[]` (always use the built-in review), or an explicit list like `["go-code-reviewer:go-review"]`.

### HTML report structure

Produce a single self-contained HTML file (all CSS inline, no external assets) so it opens cleanly from disk. This is a working review tool, so density and function come before decoration. Use the template below as the baseline and fill in every `{placeholder}`. Keep the structure and classes; add finding cards, top-action items, file rows, and diff blocks as needed.

Rules:
- Every finding is a `<details class="f ...">` card containing the real offending code (`pre.code`), a clickable `file:line` link to GitHub, the problem, and a concrete fix (`pre.code`). Keep Critical and Important cards `open`; Suggestions may be collapsed.
- The `{n_*}` counters in the nav must be the real counts, and each nav tile links to its section anchor.
- HTML-escape all code and diff text (`&` -> `&amp;`, `<` -> `&lt;`, `>` -> `&gt;`).
- In diff blocks, wrap each line: added in `<span class="a">`, removed in `<span class="d">`, hunk headers in `<span class="h">`, context lines plain.
- GitHub line link: `https://github.com/{headRepositoryOwner}/{repo}/blob/{headRefName}/{path}#L{line}`. File link in the Changed files list: same without the `#L`.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>PR Review · {title}</title>
  <style>
    :root { color-scheme: light dark;
      --bg:#f6f8fa; --card:#fff; --ink:#1f2328; --muted:#656d76; --line:#d0d7de; --accent:#0969da;
      --crit:#cf222e; --crit-bg:#ffebe9; --imp:#bc4c00; --imp-bg:#fff1e5; --sug:#1a7f37; --sug-bg:#dafbe1;
      --code:#f6f8fa; --add-bg:#e6ffec; --add-ink:#1a7f37; --del-bg:#ffebe9; --del-ink:#cf222e; }
    @media (prefers-color-scheme: dark) { :root {
      --bg:#0d1117; --card:#161b22; --ink:#e6edf3; --muted:#9198a1; --line:#30363d; --accent:#58a6ff;
      --crit:#ff7b72; --crit-bg:#2d1416; --imp:#ffa657; --imp-bg:#2a1c0f; --sug:#3fb950; --sug-bg:#11261a;
      --code:#0d1117; --add-bg:#12261a; --add-ink:#3fb950; --del-bg:#2d1416; --del-ink:#ff7b72; } }
    * { box-sizing: border-box; }
    body { font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,sans-serif; background:var(--bg); color:var(--ink); margin:0; line-height:1.55; }
    .wrap { max-width:1080px; margin:0 auto; padding:1.5rem 1.25rem 4rem; }
    a { color:var(--accent); text-decoration:none; } a:hover { text-decoration:underline; }
    code,pre,.mono { font-family:ui-monospace,SFMono-Regular,Menlo,Consolas,monospace; }
    .head { border:1px solid var(--line); border-radius:10px; background:var(--card); padding:1.1rem 1.25rem; }
    .head h1 { font-size:1.3rem; margin:.15rem 0 .6rem; }
    .head .meta { display:flex; flex-wrap:wrap; gap:.4rem .9rem; font-size:.85rem; color:var(--muted); align-items:center; }
    .head .meta b { color:var(--ink); }
    .btn { display:inline-block; background:var(--accent); color:#fff; padding:.4rem .8rem; border-radius:6px; font-weight:600; }
    .btn:hover { text-decoration:none; opacity:.92; }
    .add { color:var(--sug); } .del { color:var(--crit); } .stat { font-variant-numeric:tabular-nums; }
    .nav { display:flex; gap:.6rem; flex-wrap:wrap; margin:1rem 0; }
    .nav a { flex:1 1 110px; border:1px solid var(--line); border-radius:8px; background:var(--card); padding:.6rem .8rem; color:var(--ink); }
    .nav a:hover { text-decoration:none; border-color:var(--accent); }
    .nav .n { font-size:1.4rem; font-weight:800; line-height:1; } .nav .l { font-size:.72rem; text-transform:uppercase; letter-spacing:.04em; color:var(--muted); margin-top:.2rem; }
    .nav .crit .n { color:var(--crit); } .nav .imp .n { color:var(--imp); } .nav .sug .n { color:var(--sug); }
    h2 { font-size:1.05rem; margin:1.8rem 0 .6rem; padding-bottom:.3rem; border-bottom:1px solid var(--line); }
    .context { background:var(--card); border:1px solid var(--line); border-radius:8px; padding:1rem 1.2rem; }
    ol.top { padding-left:1.2rem; } ol.top li { margin:.35rem 0; }
    .pill { font-size:.65rem; font-weight:700; text-transform:uppercase; letter-spacing:.03em; padding:.12rem .5rem; border-radius:999px; white-space:nowrap; }
    .pill.crit { color:var(--crit); background:var(--crit-bg); } .pill.imp { color:var(--imp); background:var(--imp-bg); } .pill.sug { color:var(--sug); background:var(--sug-bg); }
    .f { border:1px solid var(--line); border-left-width:4px; border-radius:8px; background:var(--card); margin:.8rem 0; }
    .f.crit { border-left-color:var(--crit); } .f.imp { border-left-color:var(--imp); } .f.sug { border-left-color:var(--sug); }
    .f>summary { list-style:none; cursor:pointer; padding:.7rem 1rem; display:flex; gap:.6rem; align-items:baseline; }
    .f>summary::-webkit-details-marker { display:none; }
    .f .ftitle { font-weight:600; font-size:.95rem; } .f .loc { font-size:.8rem; color:var(--muted); margin-left:auto; }
    .f .body { padding:0 1rem 1rem; }
    .label { font-size:.7rem; text-transform:uppercase; letter-spacing:.05em; color:var(--muted); margin:.9rem 0 .3rem; }
    pre.code { background:var(--code); border:1px solid var(--line); border-radius:6px; padding:.7rem .9rem; overflow-x:auto; font-size:.82rem; line-height:1.45; margin:.2rem 0; white-space:pre; }
    pre.code .a { color:var(--add-ink); background:var(--add-bg); display:block; }
    pre.code .d { color:var(--del-ink); background:var(--del-bg); display:block; }
    pre.code .h { color:var(--muted); display:block; }
    .files { border:1px solid var(--line); border-radius:8px; overflow:hidden; }
    .files .row { display:flex; justify-content:space-between; gap:1rem; padding:.45rem .9rem; border-top:1px solid var(--line); font-size:.85rem; }
    .files .row:first-child { border-top:none; }
    details.diff { border:1px solid var(--line); border-radius:8px; margin:.6rem 0; background:var(--card); }
    details.diff>summary { cursor:pointer; padding:.55rem .9rem; font-size:.83rem; }
    details.diff pre.code { border:none; border-top:1px solid var(--line); border-radius:0; margin:0; }
    .foot { margin-top:2rem; color:var(--muted); font-size:.82rem; display:flex; flex-wrap:wrap; gap:.4rem 1.2rem; }
    .empty { color:var(--muted); font-style:italic; padding:.5rem 0; }
  </style>
</head>
<body><div class="wrap">

  <div class="head">
    <div class="meta"><span class="pill {verdict_class}">{verdict_label}</span><span>Pull Request Review</span></div>
    <h1>{title}</h1>
    <div class="meta">
      <span>👤 <b>{author}</b></span>
      <span>📦 <b>{repo}</b></span>
      <span class="mono">{base_ref} ← {head_ref}</span>
      <span class="stat"><span class="add">+{additions}</span> <span class="del">−{deletions}</span> · {changed_files} files</span>
      <span>🎫 {ticket_links_or_dash}</span>
      <span><a class="btn" href="{pr_url}">Open PR on GitHub ↗</a></span>
    </div>
  </div>

  <div class="nav">
    <a class="crit" href="#critical"><div class="n">{n_critical}</div><div class="l">Critical</div></a>
    <a class="imp" href="#important"><div class="n">{n_important}</div><div class="l">Important</div></a>
    <a class="sug" href="#suggestions"><div class="n">{n_suggestions}</div><div class="l">Suggestions</div></a>
  </div>

  <h2>Context</h2>
  <div class="context"><p>{2-4 sentences: what the PR does, why, what it touches, and the overall assessment}</p></div>

  <h2>Top actions</h2>
  <ol class="top">
    <li><span class="pill crit">Critical</span> {short finding} — <a href="#{anchor}">{file}:{line}</a></li>
    <!-- ranked highest priority first; link each to its finding card anchor -->
  </ol>

  <h2 id="critical">Critical</h2>
  <details class="f crit" id="{anchor}" open>
    <summary><span class="pill crit">Critical</span><span class="ftitle">{finding title}</span><span class="loc mono">{file}:{line}</span></summary>
    <div class="body">
      <div class="label">Location</div>
      <a class="mono" href="{github_line_url}">{file}:{line} ↗</a>
      <div class="label">Code</div>
      <pre class="code">{escaped offending code excerpt, the real lines}</pre>
      <div class="label">Problem</div>
      <p>{what is wrong and the concrete impact}</p>
      <div class="label">Suggested fix</div>
      <pre class="code">{escaped fix: replacement code or a diff}</pre>
    </div>
  </details>
  <!-- repeat; if none: <p class="empty">No critical issues found.</p> -->

  <h2 id="important">Important</h2>
  <details class="f imp" id="{anchor}" open> ... same inner structure ... </details>
  <!-- repeat; if none: <p class="empty">Nothing important flagged.</p> -->

  <h2 id="suggestions">Suggestions</h2>
  <details class="f sug" id="{anchor}"> ... same inner structure ... </details>
  <!-- repeat; if none: <p class="empty">No further suggestions.</p> -->

  <h2>Changed files</h2>
  <div class="files">
    <div class="row"><a class="mono" href="{github_file_url}">{path}</a><span class="stat"><span class="add">+{adds}</span> <span class="del">−{dels}</span></span></div>
    <!-- repeat per changed file -->
  </div>

  <h2>Diff</h2>
  <details class="diff">
    <summary class="mono"><b>{path}</b> <span class="stat"><span class="add">+{adds}</span> <span class="del">−{dels}</span></span></summary>
    <pre class="code"><span class="h">@@ -1,4 +1,5 @@</span><span class="d">- old line</span><span class="a">+ new line</span> context line
</pre>
  </details>
  <!-- repeat per changed file; for very large files include a truncated diff with a note and the GitHub link -->

  <div class="foot">
    <span>Reviewed {datetime}</span>
    <span><a href="{pr_url}">{pr_url}</a></span>
    <span>Tickets: {ticket_links_or_dash}</span>
  </div>

</div></body>
</html>
```

`{verdict_class}`/`{verdict_label}` summarise the PR at a glance: `crit`/"Changes needed" if any critical, else `imp`/"Review carefully" if any important, else `sug`/"Looks good". `{anchor}` is a unique slug per finding (e.g. `c1`, `i2`). Show a dash for Tickets when none are found; when `issue_tracker.base_url` is null, render ticket IDs as plain text. The emoji in the header are UI affordances, not prose, so they are fine here.

### Notification template

The Slack tool renders standard markdown (headers, bold, links). Use a title and subtitle, a short summary, a findings line, and two clickable links. The review link uses a `file://` URL so it opens the local HTML report in the browser.

```
## New PR to review
### {title}

**{author}** · `{repo}`{ticket_suffix}

{one sentence on what the PR does}

🔴 {n_critical} critical · 🟠 {n_important} important · 🟢 {n_suggestions} suggestions

📄 [Open review in browser]({file_url})  ·  🔗 [View PR on GitHub]({pr_url})
```

`{file_url}` is `file://` + the absolute report path. `{ticket_suffix}` is ` · 🎫 [{TICKET}]({base_url}{TICKET})` when a ticket is found (normalize the separator to a hyphen), or empty otherwise. Keep the summary to one sentence so the message stays scannable.

For the `webhook` notify type, POST the same data as JSON (`title, author, repo, summary, pr_url, report_path, file_url`) instead of formatting markdown.
