---
name: watch
description: Poll GitHub for new pull requests from a configured set of authors, review each new one, save an HTML report, and notify. Run "/pr-watch:watch" to start - it self-schedules on an interval. Pass a directory to use as the reviews folder, e.g. "/pr-watch:watch ~/pr-reviews".
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

# pr-watch:watch

Polls GitHub for open PRs matching a configured search, finds the ones that appeared since the last poll, spawns a review subagent for each, saves an HTML report next to the config, sends a notification, and reschedules itself.

Everything for one watcher lives in a single **reviews directory**: `config.json`, `.state.json`, and the dated HTML reports. Move or back up that one folder and you have the whole setup.

Be concise in your turn output. The work is mostly bash plus subagents; do not narrate every step.

## Step 1 - Resolve the reviews directory and load config

The reviews directory is resolved in this order:
1. The first argument to the skill, if given (e.g. `/pr-watch:watch ~/dd/tasks/pr-reviews`).
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

Call `ScheduleWakeup` with `delaySeconds` = `interval_seconds`, `prompt` = `/pr-watch:watch <REVIEWS_DIR>`, and a short `reason` such as "polling team PRs every 10m".

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

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>PR Review: {title}</title>
  <style>
    :root { color-scheme: light dark; }
    body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; max-width: 900px; margin: 2rem auto; padding: 0 1rem; line-height: 1.55; }
    header table { border-collapse: collapse; margin: 1rem 0; }
    header th { text-align: left; padding-right: 1rem; vertical-align: top; opacity: 0.7; font-weight: 600; }
    h1 { font-size: 1.5rem; }
    h2 { margin-top: 2rem; border-bottom: 1px solid currentColor; padding-bottom: 0.25rem; }
    code { background: rgba(127,127,127,0.18); padding: 0.1em 0.35em; border-radius: 4px; }
    .crit { color: #d11; } .imp { color: #c80; } .sug { opacity: 0.85; }
    ol li, ul li { margin: 0.4rem 0; }
  </style>
</head>
<body>
  <header>
    <h1>{title}</h1>
    <table>
      <tr><th>Author</th><td>{author}</td></tr>
      <tr><th>Repository</th><td><a href="{repo_url}">{repo}</a></td></tr>
      <tr><th>Pull Request</th><td><a href="{pr_url}">{pr_url}</a></td></tr>
      <tr><th>Reviewed</th><td>{datetime}</td></tr>
      <tr><th>Tickets</th><td>{ticket_links_or_dash}</td></tr>
    </table>
  </header>

  <section id="context">
    <h2>Context</h2>
    <p>{2-3 sentence summary of what the PR does, why it exists, and what it touches}</p>
  </section>

  <section id="top5">
    <h2>Top 5 Actions</h2>
    <ol>
      <li><strong class="crit">[CRITICAL]</strong> {finding} - <code>file:line</code></li>
      <!-- ranked highest priority first; use crit/imp classes -->
    </ol>
  </section>

  <section id="review">
    <h2>Full Review</h2>
    <h3 class="crit">Critical</h3>
    <ul><!-- finding, file:line, suggested fix --></ul>
    <h3 class="imp">Important</h3>
    <ul></ul>
    <h3 class="sug">Suggestions</h3>
    <ul></ul>
  </section>
</body>
</html>
```

Omit the Tickets row (or show a dash) when no tickets are found. When `issue_tracker.base_url` is null, show ticket IDs as plain text.

### Notification template

```
New PR from {author}: {title}
{one sentence on what the PR does}
PR: {pr_url}
Review: {report_path}
```
