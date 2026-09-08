---
name: handoff
description: Compact the current conversation into a handoff document for another agent to pick up.
argument-hint: "What will the next session be used for?"
disable-model-invocation: true
---

Write a handoff document summarising the current conversation so a fresh agent can continue the work.

Save it to the handoffs directory `~/Dropbox/agent-handoffs` (create it if missing), not the temporary directory and not the current workspace. This directory is agent-agnostic (handoffs may be to or from agents other than Claude), so keep the document self-contained and avoid Claude-specific assumptions where practical. Name the file `<YYYYMMDDHHMM>-<short-slug>.md`, where the prefix is the current local date-time (year, month, day, 24-hour hour, minute) so handoffs sort chronologically and the most recent is obvious, and `<short-slug>` briefly identifies the subject (e.g. `202608311144-brick-vision-handoff.md`). Get the timestamp from the system clock (e.g. `date +%Y%m%d%H%M`) rather than guessing.

Include a list of paths to github repos involved.

Include a "suggested skills" section in the document, naming which skills the next agent should call the Skill tool for.

Do not duplicate content already captured in other artifacts (specs, plans, ADRs, issues, commits, diffs). Reference them by path or URL instead.

Redact any sensitive information, such as API keys, passwords, or personally identifiable information.

If the user passed arguments, treat them as a description of what the next session will focus on and tailor the doc accordingly.

If there are older handoffs for the same subject in the directory, do not edit them. Write a new document and name the one it supersedes near the top, so the chain can be followed. Move superseded handoffs into `~/Dropbox/agent-handoffs/archive/` only when the user asks.

The counterpart skill is `pickup`, which reads a document from this directory and resumes the work.
