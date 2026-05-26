# branch-and-split

Branch the current Claude Code session and open the branch in a new horizontal terminal split. The current pane keeps the original session; the new split below starts a fresh branch (a fork of the conversation up to this point).

## Requirements

- Claude Code with the `--fork-session` flag (recent versions)
- One of: tmux, iTerm2, or Ghostty 1.3+ (macOS)

## Usage

Run `/branch-and-split` from anywhere in a Claude Code session.

The skill auto-detects the terminal in this priority order:

1. **tmux** — splits the current pane downward with `tmux split-window -v`
2. **iTerm2** — splits the current session horizontally via AppleScript
3. **Ghostty** — splits the focused terminal `direction down` via AppleScript

## How it works

Reads `$CLAUDE_CODE_SESSION_ID` from the environment and runs `claude -r <id> --fork-session` in the new pane. The fork shares the conversation history up to the moment of the split but gets a fresh session ID, so the two panes can diverge independently from there.

If `$CLAUDE_CODE_SESSION_ID` is empty, falls back to `claude --continue --fork-session`.
