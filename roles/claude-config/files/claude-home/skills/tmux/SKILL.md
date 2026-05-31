---
name: tmux
description: Inspecting and controlling other tmux sessions (capture-pane, send-keys, list-sessions). TRIGGER when the user wants to see what's happening in another tmux window/session, send a command/keystroke to a long-running session, monitor a command running elsewhere, or wake up an attached AI agent in another session. Also covers creating detached sessions for background work.
---

# tmux

By default an AI agent cannot see what's in other tmux sessions or send keys to them. **You can — via `tmux capture-pane` and `tmux send-keys`** — but the patterns aren't obvious. This skill collects the working ones.

## Discovering What's Running

```bash
# List all sessions (name, window count, attached?)
tmux ls

# List windows in a specific session
tmux list-windows -t <session-name>

# List panes within a window (each pane is one terminal)
tmux list-panes -t <session-name>:<window>     # by window name/index
tmux list-panes -a                             # across all sessions
```

Identifiers: `<session>:<window>.<pane>` — e.g. `codens-4:0.0` = session `codens-4`, window 0, pane 0. Window can be omitted if there's only one (just `codens-4`); pane defaults to 0.

## Reading What's On Screen

```bash
# Capture current visible pane content (no scrollback)
tmux capture-pane -t <session> -p

# Capture with scrollback (last N lines including history)
tmux capture-pane -t <session> -p -S -200      # last 200 lines

# Capture entire scrollback buffer
tmux capture-pane -t <session> -p -S -

# Capture a specific pane in a specific window
tmux capture-pane -t codens-4:0.0 -p -S -100

# Save capture to a file (useful for diffing or piping through grep)
tmux capture-pane -t <session> -p -S -500 > /tmp/snapshot.txt
```

`-p` means "print to stdout". Without it, the buffer is saved internally and you'd need a second command to read it. Always use `-p`.

`-S -N` = start N lines back into history. `-E -N` would set END point. `-S -` = beginning of buffer.

## Sending Commands / Keys

```bash
# Send a literal command + Enter (most common)
tmux send-keys -t <session> "ls -la" Enter

# Send a control character
tmux send-keys -t <session> C-c           # Ctrl+C
tmux send-keys -t <session> C-d           # Ctrl+D (EOF)
tmux send-keys -t <session> Escape        # Esc
tmux send-keys -t <session> Up Up Enter   # arrow keys

# Send multi-line input — issue separate send-keys per line
tmux send-keys -t <session> "line1" Enter "line2" Enter

# Quick: send Enter only (e.g. wake up a paused REPL)
tmux send-keys -t <session> Enter
```

**Quote handling**: arguments are joined with spaces by tmux. To preserve quoting in target shell, use:
```bash
tmux send-keys -t <session> 'echo "hello world"' Enter
```

## Common Workflow — Inspect a Long-Running Job

```bash
# 1. find the session
tmux ls | grep -i build

# 2. peek at last 50 lines without disturbing it
tmux capture-pane -t build -p -S -50

# 3. (optional) send a status query
tmux send-keys -t build "" Enter        # nudge a frozen prompt
```

## Common Workflow — Wake Up Another AI Agent

When another Claude session is running in tmux waiting for user input:

```bash
# See what it's waiting on
tmux capture-pane -t codens-4 -p -S -30

# Send a prompt + Enter
tmux send-keys -t codens-4 "続けて" Enter

# If it's in a paused/idle state, may need to nudge
tmux send-keys -t codens-4 Enter
```

## Common Workflow — Run Something in Background Session

```bash
# Create detached session running a command
tmux new-session -d -s long-build "cd /repo && make all"

# Check on it later
tmux capture-pane -t long-build -p -S -100

# Kill when done
tmux kill-session -t long-build
```

## Pitfalls

- **`-t` is mandatory** when there are multiple sessions / windows. tmux's "current" defaults bite you.
- **`tmux send-keys "ls"` (without Enter) just types `ls`** but doesn't run it. Always append `Enter`.
- **Capture-pane shows what's currently on the visible pane**, including any partially-typed input. To see only "stable" history, scroll back with `-S -50` or similar.
- **Multiple capture in a row is cheap** — don't worry about doing 3-4 captures while debugging.
- **Don't `kill-session` without confirming**: `tmux ls` first to make sure you target the right one.
- **Don't send Ctrl+C blindly**: capture-pane first, see what's running, decide whether interrupting is safe.

## Anti-Patterns

- **Polling every 1 second** to watch progress: capture every 30-60s instead, or use the `Monitor` tool with a tail-f-like script.
- **Sending many keys quickly**: tmux can drop characters under heavy bursts. Add `\;` separators or put each in it's own `send-keys` call.
- **Trusting capture-pane to be "current"**: terminal apps sometimes redraw async. If you sent a command, give it 1-2 seconds before capturing the result.
