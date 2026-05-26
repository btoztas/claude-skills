---
name: branch-and-split
description: Branch the current Claude Code session and open the branch(es) in new horizontal terminal split(s). Usage - "/bg:branch-and-split" for one split, or "/bg:branch-and-split N" for N independent forks (default 1, capped at 10). Supports tmux, iTerm2, and Ghostty.
user_invocable: true
---

# branch-and-split

Branches the current session into N new horizontal terminal split(s). Each split is an independent fork of the conversation up to this point. Be silent — no preamble, no progress, no summary. Run one Bash call, then stop.

## Parse args

The first arg is N (number of splits). Default to `1` if missing, non-numeric, less than 1, or greater than 10. Substitute the parsed value for `<N>` in the bash below.

## The one Bash call

Open N horizontal splits as siblings of the current pane (not nested). Priority: tmux > iTerm2 > Ghostty.

```bash
N=<N>
CMD="claude -r ${CLAUDE_CODE_SESSION_ID:-} --fork-session"
[ -z "$CLAUDE_CODE_SESSION_ID" ] && CMD="claude --continue --fork-session"

if [ -n "$TMUX" ]; then
  ORIG=$(tmux display-message -p '#{pane_id}')
  for _ in $(seq 1 "$N"); do
    tmux split-window -v -t "$ORIG" "$CMD" >/dev/null 2>&1
  done
  tmux select-layout even-vertical >/dev/null 2>&1
elif [ "$TERM_PROGRAM" = "iTerm.app" ]; then
  osascript \
    -e "tell application \"iTerm2\"" \
    -e "  set origSession to current session of current window" \
    -e "  repeat $N times" \
    -e "    tell origSession" \
    -e "      set newSession to (split horizontally with default profile)" \
    -e "      tell newSession to write text \"$CMD\"" \
    -e "    end tell" \
    -e "  end repeat" \
    -e "end tell" >/dev/null 2>&1
elif [ "$TERM_PROGRAM" = "ghostty" ]; then
  osascript \
    -e "tell application \"Ghostty\"" \
    -e "  set cfg to new surface configuration" \
    -e "  set initial input of cfg to (\"$CMD\" & return)" \
    -e "  set origTerm to focused terminal of selected tab of front window" \
    -e "  repeat $N times" \
    -e "    split origTerm direction down with configuration cfg" \
    -e "  end repeat" \
    -e "end tell" >/dev/null 2>&1
else
  echo "Unsupported terminal. Run manually in $N new split(s): $CMD" >&2
  exit 1
fi
```

## Output rules

- On success: output nothing. End the turn.
- On failure (non-zero exit): output only the one-line error.
