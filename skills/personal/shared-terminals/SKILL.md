---
name: shared-terminals
description: Run shell commands in user-attachable tmux shells via the claude-term wrapper, so the user can supervise and interact at any time. Use whenever executing commands with side effects, running AWS/cloud CLIs, starting servers or REPLs, or when the user mentions tmux, shared terminals, or wanting to watch/take over a shell.
---

# Shared terminals (claude-term)

## The rule

Justin requires that every shell doing real work is one he can attach to and
supervise or take over at any moment. The problem is not that Claude runs
commands — it's that they must never run somewhere invisible or untouchable.

This applies to **every shell command, not just ones with side effects.**
`git status`, `ls`, `find`, `cat` — anything typed into a shell — goes
through the shared session too, so it's visible in the same place as
everything else, not scattered between a hidden shell and tmux windows.

So: **every shell command runs in a named tmux window inside this client's
tmux session**, managed via the wrapper:

```
~/.claude/bin/claude-term
```

**Each Claude client gets its own session** so concurrent clients never
clobber each other's shells. The session is `claude-<sid8>` — the first 8
chars of `CLAUDE_CODE_SESSION_ID`, the stable per-client id Claude Code
exports to every shell (it falls back to bare `claude` only if that id is
absent). You never type the name; the wrapper derives it. To tell Justin how
to attach, run `claude-term attach` — it prints the exact command for *this*
client's session. `claude-term sessions` lists every `claude-*` client
session on the tmux server (with its root dir and idle age) so he can pick
which one to attach to.

Claude's own Bash tool shell is used **only** to invoke `claude-term` (or raw
`tmux` against the shared session — see the escape hatch below). It never
runs git, file, or any other shell command directly, including read-only
inspection. For file reads/writes that don't need a shell, use the dedicated
Read/Edit/Write tools instead of Bash entirely — those aren't shell commands
and this rule doesn't apply to them.

## Why this design

- Justin authenticates to AWS with Granted (`assume`) — SSO tokens live in the
  macOS Keychain and every profile uses `credential_process`, so credentials
  work in **any** shell without re-auth. Agent shells have no `aws` alias
  (dotfiles drop the 1Password op wrapper when `CLAUDE_TERM` is set); always
  pass `--profile <name>` explicitly since `AWS_PROFILE` is only exported in
  the window where Justin ran `assume`.
- He may need to answer a prompt, enter a code, or kill a command at any time:
  attaching to this client's session (`claude-term attach` prints the command)
  puts him in front of every shell it is using. Other Claude clients live in
  their own sessions — isolated, not hijacked.
- Multiple shells are fine — one window per concern (e.g. `aws`, `server`,
  `build`) — as long as they all live in the shared session.

## Command reference

```
claude-term list                          # shells in THIS client's session + which one the user views
claude-term sessions                      # all claude-* client sessions (root, idle age, current/stale)
claude-term reap-sessions                 # kill orphaned claude sessions (idle+unattached; never current)
claude-term new  <name> [dir]             # create shell <name> (auto-created by run too)
claude-term run  <name> [-t sec] [-f] -- CMD
                                          # run CMD, wait, print output; exits with CMD's
                                          # exit code, 124 on timeout (default 120s),
                                          # 2 if the window is busy in an interactive
                                          # program (-f overrides)
claude-term send <name> KEYS...           # raw send-keys: text, Enter, C-c, Up, etc.
claude-term read <name> [-n N | -a]       # screen contents (last N lines / all history)
claude-term kill <name>                   # close a shell
claude-term cleanup                       # close idle agent-owned shells (unpinned, at a
                                          # shell prompt, untouched 60+ min)
claude-term cleanup-all                   # close every unpinned shell except window 0
claude-term stale-status                  # report whether terminal activity is stale
claude-term pin <name>                    # protect a shell from cleanup/cleanup-all
claude-term unpin <name>                  # release a pinned shell
claude-term attach                        # prints the attach command for the user
```

While a window is actively being driven by `run` or `send`, the wrapper
renames it to `#> <name>` so Justin can see at a glance which shells are
live versus idle when he attaches. `run` restores the base name when the
command completes (including after a timeout — it stays marked, since the
command may still be running). A `send`-driven window (REPL, SSH, wizard)
stays marked until a later `list` or `read` notices it has returned to a
shell prompt and restores it. Window 0 is never renamed. All commands
accept either the base name or the `#> `-prefixed name.

## How to work

- **Batch commands**: `claude-term run <shell> -- '<command>'`. Quote the
  command as one argument when it has pipes, quotes, or redirects. Check the
  exit code as usual. `run` enforces **one line, max 500 chars** (env
  `CLAUDE_TERM_MAXLEN` adjusts; `-f` skips the length check only — newlines
  are always fatal because they split the command and orphan the markers).
- **Bigger than one line?** Two options, by intent:
  - Need exit codes / output per step, or it's over the length limit → write
    a temp script (scratchpad dir), then `run <shell> -- 'zsh /path/to/script'`.
  - Just need a sequence typed in order (setup steps, priming a REPL) →
    multi-line `send` is a *feature*: every line executes sequentially, like
    pasting a block into the terminal. No exit codes, so `read` the screen
    afterwards to confirm state.
- **Long-running commands**: pass a generous `-t`, or start it, then poll with
  `claude-term read`. On timeout (exit 124) the command is still running —
  don't resend it; read the window later.
- **Interactive programs** (REPLs, ssh, debuggers, wizards): never use `run`
  (its completion markers need a shell prompt). Drive them with `send` and
  inspect with `read`. `run` checks the window's foreground process and
  refuses (exit 2) if it isn't a shell; `run -f` overrides for the ssh case
  where the remote end really is at a shell prompt. The guard is a snapshot —
  still `read` a window first if its state is uncertain.
- **Servers/watchers**: give them their own window and leave it running;
  monitor with `read`.
- **Naming**: default to one reusable shell (e.g. `work`) for ordinary
  serialized work — inspection, git, builds, tests, cloud CLIs. Reuse it
  across concerns rather than creating a new window just because the task
  changed; a shell's name is not a task boundary. Only open another window
  when something must run concurrently: a server/watcher that must keep
  running while another shell interacts with it, two commands needing
  different AWS/account auth at the same time, or an interactive
  prompt/REPL/SSH session that must stay live while other work continues.
  `kill` windows once a concurrent need ends.
- **One window per AWS account**: exports persist in a window's shell, so
  give each target account its own window, set it up once, and `pin` it so
  idle cleanup won't close it:
  `run aws-<account> -- 'export AWS_PROFILE="<profile>" AWS_REGION=<region>'`
  then `pin aws-<account>`. After that, plain `aws` commands in that window
  hit that account, and several windows can work different accounts
  concurrently (the Keychain SSO token covers the whole org; no re-auth to
  add another account). `command aws configure list-profiles` shows the
  available profiles. `unpin` and close the window once the concurrent need
  ends.
- **Auth handoffs**: when a command needs interactive auth (browser SSO,
  1Password approval, MFA), run it in a shell, then tell Justin which window
  needs him (`claude-term list` shows names; he attaches and switches with
  `Ctrl-b n` or `Ctrl-b w`). Wait for him to confirm before proceeding.
- **Shell lifecycle**: `new`/`run`/`send`/`pin`/`unpin`/`cleanup`/
  `cleanup-all` all mark terminal activity. Every 10th such call (env
  `CLAUDE_TERM_CLEANUP_EVERY`), idle agent-owned shells — unpinned, sitting
  at a shell prompt, untouched for 60+ minutes (`CLAUDE_TERM_IDLE_MINUTES`)
  — are closed automatically; this never touches window 0 or pinned windows.
  On the first terminal interaction after resuming a session, run
  `claude-term stale-status` before anything else. If it reports `stale`
  (8+ hours since any terminal activity, `CLAUDE_TERM_STALE_HOURS`), tell
  Justin old windows may still be open and ask before running
  `claude-term cleanup-all` — it's a confirmed, explicit cleanup, not
  something to run unprompted.

## Escape hatch: raw tmux

The invariant is a **user-attachable tmux session**, not the wrapper.
`claude-term` covers the common cases, but when its verbs don't fit, use
`tmux` directly against windows of this client's session — this is expected,
not a violation. (Need the session name for a raw `tmux -t` command?
`claude-term attach` prints it, or read `$CLAUDE_TERM_SESSION` if you set it.)
Typical reasons:

- `tmux capture-pane -J` to re-join wrapped long lines, or `-e` to keep colors
- `tmux pipe-pane` to stream a window's output to a log file (better than
  polling for long REPL/server sessions)
- `tmux respawn-window` to revive a dead shell in place; `resize-window` etc.
- REPL work generally: `claude-term send`/`read` are thin passthroughs to
  `send-keys`/`capture-pane`, so use whichever is clearer — flags like
  `send -l` (literal text) pass straight through.

What is **not** allowed: creating sessions/windows outside this client's
session, or running workloads in your own Bash shell. Everything must stay
where the user can attach and see it.

## Safety

- **Never print credential values.** Do not dump `AWS_*` (or similar) env var
  values into captured output — names only (`env | cut -d= -f1`) if inspection
  is needed. Captured screens land in the transcript.
- The user may be typing in any window at any time. Before `send`-ing to a
  window you haven't touched recently, `read` it first to see its state.
- Window 0 (`zsh`) is Justin's own working shell — read it if useful, but
  don't run commands there uninvited; use your own named windows.
