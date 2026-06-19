# bg

Bruno's personal Claude Code skills.

## Skills

### branch-and-split

Branch the current Claude Code session and open the branch in a new horizontal terminal split. The current pane keeps the original session; the new split below starts a fresh branch (a fork of the conversation up to this point).

**Requirements:**

- Claude Code with the `--fork-session` flag (recent versions)
- One of: tmux, iTerm2, or Ghostty 1.3+ (macOS)

**Usage:**

- `/bg:branch-and-split` — open one split with a fork
- `/bg:branch-and-split N` — open N splits, each an independent fork (default 1, max 10)

The skill auto-detects the terminal in this priority order:

1. **tmux** — splits the current pane downward with `tmux split-window -v`
2. **iTerm2** — splits the current session horizontally via AppleScript
3. **Ghostty** — splits the focused terminal `direction down` via AppleScript

## How it works

Reads `$CLAUDE_CODE_SESSION_ID` from the environment and runs `claude -r <id> --fork-session` in the new pane. The fork shares the conversation history up to the moment of the split but gets a fresh session ID, so the two panes can diverge independently from there.

If `$CLAUDE_CODE_SESSION_ID` is empty, falls back to `claude --continue --fork-session`.

### pr-watch

Watch GitHub for new pull requests from a set of authors, review each new one automatically, save an HTML report, and send a notification. The watcher reschedules itself on an interval, so you start it once and it keeps polling.

**Usage:**

- `/bg:pr-watch` — start watching, using the default reviews folder (`~/pr-reviews`)
- `/bg:pr-watch ~/some/dir` — use a specific reviews folder

On each run it loads its config, searches GitHub with your query, finds PRs that appeared since the last poll, spawns one review subagent per new PR (each checks the PR out in a git worktree, reviews it with an installed review skill or a built-in review, and writes an HTML report), sends a brief notification, and schedules the next poll.

**Reviews directory.** Everything for one watcher lives in a single folder: `config.json`, `.state.json`, and the dated HTML reports. The folder is resolved from the argument, then `$PR_WATCH_DIR`, then the `~/pr-reviews` default. Because the self-scheduled poll re-passes the directory, the location you start with sticks.

**Configuration** lives in `<reviews_dir>/config.json` and is created on first run (or written by hand):

```json
{
  "search": {
    "type": "github_rest",
    "query": "org:ORG is:pr is:open -is:draft author:USER1 author:USER2",
    "per_page": 100
  },
  "interval_seconds": 600,
  "repo_base_path": "~/src",
  "review": { "model": "claude-opus-4-8", "effort": "high", "skills": "auto" },
  "issue_tracker": { "pattern": "[A-Z]+-[0-9]+", "base_url": "https://your-org.atlassian.net/browse/" },
  "notify": { "type": "slack", "slack_recipient": "Your Name", "webhook_url": null }
}
```

- `search.query` is the part after `q=` in a GitHub PR search.
- `repo_base_path` is where local clones live; missing repos are cloned on demand.
- `review.skills` accepts `"auto"` (discover and pick an installed review skill per language), `[]` (always use the built-in review), or an explicit list like `["go-code-reviewer:go-review"]`.
- `issue_tracker` is optional; set `base_url` to `null` to show ticket IDs without links.
- `notify.type` is `slack`, `webhook`, or `none`.

**Requirements:** `gh` (authenticated) and `jq`. Slack notifications need the Slack MCP plugin. Review skills are optional - with none installed, the built-in review still runs.
