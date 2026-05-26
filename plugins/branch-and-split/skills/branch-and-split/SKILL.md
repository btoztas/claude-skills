---
name: branch-and-split
description: Branch the current Claude Code session and open the branch in a new Ghostty horizontal split. Current terminal keeps the original session; the new split starts the branch.
user_invocable: true
---

# branch-and-split

Branches the current session into a new Ghostty horizontal split. Be silent — no preamble, no progress, no summary. Run one Bash call, then stop.

## The one Bash call

Detect the terminal environment and open a horizontal split (new pane below the current one). Priority: tmux > iTerm2 > Ghostty.

```bash
CMD="claude -r ${CLAUDE_CODE_SESSION_ID:-} --fork-session"
[ -z "$CLAUDE_CODE_SESSION_ID" ] && CMD="claude --continue --fork-session"

if [ -n "$TMUX" ]; then
  tmux split-window -v "$CMD" >/dev/null 2>&1
elif [ "$TERM_PROGRAM" = "iTerm.app" ]; then
  osascript \
    -e "tell application \"iTerm2\"" \
    -e "  tell current session of current window" \
    -e "    set newSession to (split horizontally with default profile)" \
    -e "    tell newSession to write text \"$CMD\"" \
    -e "  end tell" \
    -e "end tell" >/dev/null 2>&1
elif [ "$TERM_PROGRAM" = "ghostty" ]; then
  osascript \
    -e "tell application \"Ghostty\"" \
    -e "  set cfg to new surface configuration" \
    -e "  set initial input of cfg to (\"$CMD\" & return)" \
    -e "  set currentTerm to focused terminal of selected tab of front window" \
    -e "  split currentTerm direction down with configuration cfg" \
    -e "end tell" >/dev/null 2>&1
else
  echo "Unsupported terminal. Run manually in a new split: $CMD" >&2
  exit 1
fi
```

## Output rules

- On success: output nothing. End the turn.
- On failure (non-zero exit): output only the one-line error.
