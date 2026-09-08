---
name: pickup
description: Resume work from a handoff document written by a previous agent session.
argument-hint: "Optional: a slug, keyword, or filename to narrow the list."
disable-model-invocation: true
---

Resume work from a handoff document. This is the counterpart to `handoff`, which writes the documents this skill reads.

## Find the document

Handoffs live in `~/Dropbox/agent-handoffs`. Files are named `<YYYYMMDDHHMM>-<short-slug>.md`, so a plain sort puts the most recent last. Ignore the `archive/` subdirectory unless the user names a file in it.

Always let the user choose. Never pick a document on their behalf, even when only one looks relevant.

- With no arguments: list the ten most recent files, newest first.
- With arguments: treat them as a search. Match against filenames first, then against the first heading of each document. List up to ten matches, newest first. If nothing matches, say so and fall back to the ten most recent.

Present the list as a numbered menu, one line per file: the date and time from the filename prefix, formatted for reading (e.g. `2026-09-07 22:21`), then the first heading of the document. Read only the first heading of each file at this stage, not the whole document. Where the harness offers a structured choice prompt, use it; otherwise ask in plain text. Wait for the answer.

Once a document is chosen:

- If it names one it supersedes, the chosen one is authoritative; the older one is background only.
- If a newer handoff shares the same slug, point that out and confirm the user still wants the older one.

Read the whole chosen document before doing anything else.

## Check the ground truth

A handoff describes the world as it was when it was written. Before acting on it, confirm the parts that could have moved:

- Each repository it lists: does the path exist, what branch is checked out, is the working tree clean, and are there commits since the handoff was written (compare the file's timestamp prefix against `git log`).
- Each issue, PR, plan, spec, or notebook it references by path or URL: does it still exist, and has its status changed.
- Each item under "next steps" or "not verified": has it already been done. A commit message, closed issue, or changed file is evidence; do not assume.
- Any process it says may still be running (servers, watchers, tmux windows): check, do not restart blindly.

If the handoff is more than a few days older than the newest activity in its repositories, treat its "next steps" as a hypothesis, not a plan, and say so.

## Load the suggested skills

The document carries a "suggested skills" section. For each model-invoked skill it names, call the Skill tool with that skill before starting the work it applies to. For any skill it names that turns out to be user-invoked, tell the user to run it by name instead. If the handoff says a skill is mandatory for this environment (for example a shell-supervision skill), load it first, before any shell command.

## Brief the user, then start

Give a short brief before touching anything: what the handoff says the work is, what state it claims, what you found had changed since, and the first step you intend to take. Keep it to a few sentences or a short list. Flag any next step that needs the user (manual QA, credentials, a decision) rather than attempting it.

Then begin with the first next step that does not need the user. Do not wait for permission on steps the handoff already lays out; the user invoked this skill to resume the work, not to be asked whether to.

## Closing out

Do not modify or delete the handoff document. When the work it describes is complete, tell the user it can be moved into `~/Dropbox/agent-handoffs/archive/`, and move it only if they ask. If the session ends with work still open, the user can run `handoff` again to write a fresh document that supersedes this one.

## Keep it agent-agnostic

The handoff directory is shared with agents other than Claude. Do not assume the document was written by the same harness that is reading it. Where a document uses a term this harness lacks, translate it to the nearest local equivalent rather than treating it as an error.
