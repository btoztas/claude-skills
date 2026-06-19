# pr-watch

Watch GitHub for new pull requests from a set of authors, review each new one automatically, save an HTML report, and send a notification. The watcher reschedules itself on an interval, so you start it once and it keeps polling.

## Skills

### watch

`/pr-watch:watch [reviews_dir]`

On each run it:

1. Loads config from the reviews directory (runs a short interactive setup the first time).
2. Searches GitHub with your configured query and finds PRs that appeared since the last poll.
3. Spawns one review subagent per new PR (parallel). Each subagent checks the PR out in a git worktree, reviews it with an installed review skill (or a built-in review), and writes an HTML report.
4. Sends a brief notification with links to the report and the PR.
5. Reschedules the next poll.

## Reviews directory

Everything for one watcher lives in a single folder: `config.json`, `.state.json`, and the dated HTML reports. The folder is resolved in this order:

1. The argument you pass, e.g. `/pr-watch:watch ~/pr-reviews`.
2. The `$PR_WATCH_DIR` environment variable.
3. Default: `~/pr-reviews`.

Because the self-scheduled poll re-passes the directory each time, the location you start with sticks.

## Configuration

`config.json` lives in the reviews directory. It is created on first run; you can also write it by hand.

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

Notes:

- `search.query` is the part after `q=` in a GitHub PR search. Build it on github.com and copy the query text.
- `repo_base_path` is where local clones live. Missing repos are cloned on demand.
- `review.skills` accepts `"auto"` (discover and pick an installed review skill per language), `[]` (always use the built-in review), or an explicit list like `["go-code-reviewer:go-review"]`.
- `issue_tracker` is optional. Set `base_url` to `null` to show ticket IDs without links, or drop the section entirely.
- `notify.type` is `slack`, `webhook`, or `none`. Slack uses the Slack MCP plugin; webhook POSTs a small JSON payload.

## Requirements

- `gh` CLI, authenticated.
- `jq`.
- For Slack notifications: the Slack MCP plugin connected.
- Review skills are optional. With none installed, the built-in review still runs.
