# Personal

Justin's own skills, kept in this fork only. They are specific to one machine and workflow (Dropbox paths, a tmux wrapper, a Hermes deployment) and are not intended for upstream. They are excluded from the plugin and the top-level README, get no docs pages, and the repo's prose rules for promoted skills do not apply here.

`scripts/link-skills.sh` links them into `~/.claude/skills` and `~/.agents/skills` alongside everything else.

- **[pickup](./pickup/SKILL.md)**: Resume work from a handoff document in `~/Dropbox/agent-handoffs`. Counterpart to `handoff`. User-invoked.
- **[shared-terminals](./shared-terminals/SKILL.md)**: Run every shell command in a user-attachable tmux session via the `claude-term` wrapper, so the user can watch or take over any shell.
- **[hermes-upgrade](./hermes-upgrade/SKILL.md)**: Drive a Hermes agent upgrade, patched-fork cycle, or clean-slate rebuild using the blue/green tooling in the Hermes config backup.
