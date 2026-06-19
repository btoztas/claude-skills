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
>    where `<slug>` is the first ~5 words of the title, lowercased and kebab-cased (strip the ticket prefix and punctuation). Use today's date. The report must include: a Context section (2-3 sentences on what the PR does and why), a Top 5 Actions list (the highest-priority findings), and the Full Review grouped into Critical / Important / Suggestions. Follow these writing conventions: friendly and clear, no emojis, no em-dashes, do not assume the PR is production-ready, hyperlink any tickets.
>
> 6. Clean up the worktree: `git -C "$REPO_PATH" worktree remove "$WT" --force` and `git -C "$REPO_PATH" branch -D pr-<number>` (ignore errors).
>
> 7. Notify, based on the notify type:
>    - `slack`: find the recipient with `mcp__plugin_slack_slack__slack_search_users` (query = the configured recipient name), then `mcp__plugin_slack_slack__slack_send_message` to their DM. Keep it very brief - see the skill's notification template.
>    - `webhook`: `curl -fsS -X POST -H 'Content-Type: application/json' -d '<json>' "<webhook_url>"` with fields `{title, author, repo, summary, pr_url, report_path}`.
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

Produce a single self-contained HTML file (all CSS inline, no external assets) so it opens cleanly from disk. Aim for a fresh, scannable, modern look that makes the review inviting. Use the template below as the baseline and fill in every `{placeholder}`. Keep the structure and classes; you may extend the content but do not strip the styling.

Fill the four `{n_*}` counters with the real number of findings in each bucket. Repeat the finding-card and `top5` `<li>` blocks as needed. Use the severity classes (`crit`, `imp`, `sug`) consistently.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>PR Review · {title}</title>
  <style>
    :root {
      color-scheme: light dark;
      --bg: #f6f8fa; --card: #ffffff; --ink: #1f2328; --muted: #656d76; --line: #d0d7de;
      --accent: #6e40c9; --accent2: #4493f8;
      --crit: #cf222e; --crit-bg: #ffebe9; --imp: #bc4c00; --imp-bg: #fff1e5;
      --sug: #1a7f37; --sug-bg: #e6f4ea; --add: #1a7f37; --del: #cf222e;
    }
    @media (prefers-color-scheme: dark) {
      :root {
        --bg: #0d1117; --card: #161b22; --ink: #e6edf3; --muted: #9198a1; --line: #30363d;
        --accent: #bc8cff; --accent2: #58a6ff;
        --crit: #ff7b72; --crit-bg: #2d1416; --imp: #ffa657; --imp-bg: #2a1c0f;
        --sug: #3fb950; --sug-bg: #11261a; --add: #3fb950; --del: #ff7b72;
      }
    }
    * { box-sizing: border-box; }
    body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
      background: var(--bg); color: var(--ink); margin: 0; line-height: 1.6;
      -webkit-font-smoothing: antialiased; }
    .wrap { max-width: 880px; margin: 0 auto; padding: 2rem 1.25rem 4rem; }
    .hero { background: linear-gradient(135deg, var(--accent), var(--accent2));
      border-radius: 16px; padding: 1.75rem 1.5rem; color: #fff; box-shadow: 0 8px 30px rgba(0,0,0,0.12); }
    .hero .eyebrow { font-size: .72rem; letter-spacing: .08em; text-transform: uppercase; opacity: .85; margin: 0 0 .4rem; }
    .hero h1 { font-size: 1.55rem; margin: 0 0 1rem; line-height: 1.3; }
    .chips { display: flex; flex-wrap: wrap; gap: .5rem; margin-bottom: 1.25rem; }
    .chip { background: rgba(255,255,255,0.18); border-radius: 999px; padding: .25rem .7rem; font-size: .82rem; backdrop-filter: blur(4px); }
    .chip b { font-weight: 700; }
    .cta { display: inline-flex; align-items: center; gap: .4rem; background: #fff; color: #1f2328;
      font-weight: 600; text-decoration: none; padding: .55rem 1.1rem; border-radius: 8px; font-size: .9rem; box-shadow: 0 2px 8px rgba(0,0,0,.15); }
    .cta:hover { transform: translateY(-1px); }
    .scoreboard { display: flex; gap: .75rem; flex-wrap: wrap; margin: 1.5rem 0; }
    .score { flex: 1 1 120px; background: var(--card); border: 1px solid var(--line); border-radius: 12px; padding: .9rem 1rem; }
    .score .n { font-size: 1.7rem; font-weight: 800; line-height: 1; }
    .score .l { font-size: .78rem; color: var(--muted); margin-top: .3rem; text-transform: uppercase; letter-spacing: .04em; }
    .score.crit .n { color: var(--crit); } .score.imp .n { color: var(--imp); } .score.sug .n { color: var(--sug); }
    .card { background: var(--card); border: 1px solid var(--line); border-radius: 12px; padding: 1.25rem 1.4rem; margin: 1rem 0; }
    h2 { font-size: 1.15rem; margin: 2rem 0 .25rem; }
    .sub { color: var(--muted); font-size: .85rem; margin: 0 0 .75rem; }
    a { color: var(--accent2); }
    code { font-family: ui-monospace, SFMono-Regular, Menlo, monospace; font-size: .85em;
      background: rgba(127,127,127,0.16); padding: .12em .4em; border-radius: 5px; }
    .meta { display: flex; flex-wrap: wrap; gap: .35rem 1.25rem; color: var(--muted); font-size: .85rem; margin-top: 1rem; }
    .meta a { color: var(--muted); }
    ol.top5 { list-style: none; counter-reset: t; padding: 0; margin: 0; }
    ol.top5 li { counter-increment: t; position: relative; padding: .8rem 1rem .8rem 3rem; border: 1px solid var(--line);
      border-radius: 10px; margin-bottom: .6rem; background: var(--card); }
    ol.top5 li::before { content: counter(t); position: absolute; left: .9rem; top: .8rem;
      width: 1.5rem; height: 1.5rem; border-radius: 50%; background: var(--accent); color: #fff;
      font-size: .85rem; font-weight: 700; display: grid; place-items: center; }
    .pill { display: inline-block; font-size: .68rem; font-weight: 700; letter-spacing: .03em; text-transform: uppercase;
      padding: .12rem .5rem; border-radius: 999px; vertical-align: middle; margin-right: .4rem; }
    .pill.crit { color: var(--crit); background: var(--crit-bg); }
    .pill.imp { color: var(--imp); background: var(--imp-bg); }
    .pill.sug { color: var(--sug); background: var(--sug-bg); }
    .finding { border-left: 4px solid var(--line); padding: .25rem 0 .25rem 1rem; margin: 1rem 0; }
    .finding.crit { border-color: var(--crit); } .finding.imp { border-color: var(--imp); } .finding.sug { border-color: var(--sug); }
    .finding h4 { margin: 0 0 .3rem; font-size: .98rem; }
    .finding .loc { font-size: .8rem; color: var(--muted); }
    .finding .fix { margin-top: .5rem; }
    .empty { color: var(--muted); font-style: italic; }
  </style>
</head>
<body>
  <div class="wrap">
    <div class="hero">
      <p class="eyebrow">Pull Request Review</p>
      <h1>{title}</h1>
      <div class="chips">
        <span class="chip">👤 <b>{author}</b></span>
        <span class="chip">📦 {repo}</span>
        <span class="chip" style="color:#caffbf">+{additions}</span>
        <span class="chip" style="color:#ffc9c9">-{deletions}</span>
        <span class="chip">📄 {changed_files} files</span>
      </div>
      <a class="cta" href="{pr_url}">Open PR on GitHub →</a>
    </div>

    <div class="scoreboard">
      <div class="score crit"><div class="n">{n_critical}</div><div class="l">Critical</div></div>
      <div class="score imp"><div class="n">{n_important}</div><div class="l">Important</div></div>
      <div class="score sug"><div class="n">{n_suggestions}</div><div class="l">Suggestions</div></div>
    </div>

    <h2>Context</h2>
    <div class="card">
      <p>{2-3 sentence summary of what the PR does, why it exists, and what it touches}</p>
    </div>

    <h2>Top 5 Actions</h2>
    <p class="sub">The highest-impact things to look at first.</p>
    <ol class="top5">
      <li><span class="pill crit">Critical</span>{finding} <code>file:line</code></li>
      <!-- repeat, ranked highest priority first; use crit/imp/sug pills -->
    </ol>

    <h2>Full Review</h2>

    <h3>🔴 Critical</h3>
    <div class="finding crit">
      <h4>{finding title}</h4>
      <div class="loc"><code>file:line</code></div>
      <p>{what is wrong and why it matters}</p>
      <div class="fix">{suggested fix}</div>
    </div>
    <!-- repeat; if none: <p class="empty">No critical issues found.</p> -->

    <h3>🟠 Important</h3>
    <div class="finding imp"> ... </div>
    <!-- repeat; if none: <p class="empty">Nothing important flagged.</p> -->

    <h3>🟢 Suggestions</h3>
    <div class="finding sug"> ... </div>
    <!-- repeat; if none: <p class="empty">No further suggestions.</p> -->

    <div class="meta">
      <span>Reviewed {datetime}</span>
      <span><a href="{pr_url}">{pr_url}</a></span>
      <span>Tickets: {ticket_links_or_dash}</span>
    </div>
  </div>
</body>
</html>
```

The emoji in the chips and section headers are part of the report UI (they help scanning), not documentation prose, so they are fine here. Show a dash for Tickets when none are found; when `issue_tracker.base_url` is null, render ticket IDs as plain text.

### Notification template

```
New PR from {author}: {title}
{one sentence on what the PR does}
PR: {pr_url}
Review: {report_path}
```
