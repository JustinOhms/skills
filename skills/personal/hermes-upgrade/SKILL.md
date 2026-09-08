---
name: hermes-upgrade
description: Perform a Hermes agent upgrade / patched-fork cycle / clean-slate rebuild as Claude Code (NOT as Hermes updating itself). Use when Justin — or Hermes delegating to Claude — asks to upgrade Hermes, catch the fork up with upstream, deploy/rebuild a feature, or recover a broken deployment. Drives the blue/green tooling in ~/Dropbox/Dev/hermes-config-backup/rescue/.
---

# Hermes upgrade (Claude executes, not Hermes)

## Core principle
A system must never perform surgery on its own running brain. **Claude Code runs
the upgrade in a supervised shell, independent of the Hermes gateway process** —
so restarting/rebuilding slots can never kill the upgrader. Hermes only *triggers*
(delegates intent); it never updates itself in place. Read ADR-0042 (deploy
architecture) and ADR-0043 (migration/model) — snapshots in
`~/Dropbox/Dev/hermes-config-backup/adr-snapshots/` — before acting.

## Front doors — the upgrade funnel (all roads lead here)
Every way to request an upgrade resolves to the SAME process (this cycle, run by an
independent coding agent). The single chokepoint is
`~/Dropbox/Dev/hermes-config-backup/rescue/hermes-upgrade-handoff`:
- **Hermes `/upgrade` command** (planned) or **NL "upgrade yourself"** → the
  `hermes-self-upgrade` *Hermes* skill runs `hermes-upgrade-handoff` (Hermes never
  self-updates).
- **Bare CLI** → run `hermes-upgrade-handoff`.
- **You (Claude/Codex) started directly in the working copy** → you ARE the agent;
  just run this cycle in-session.

`hermes-upgrade-handoff` spawns `claude`|`codex` **detached in tmux session
`hermes-upgrade`** (swap-safe + attachable), seeded with "run the hermes-upgrade
cycle." Agent = prompt (2-min timeout → configurable default `claude`, fallback
`codex`). **If you were spawned by it, just proceed with the procedure below** — and
still honor the two human gates (the user attaches to `tmux attach -t hermes-upgrade`
to approve). Codex has no Claude skills, so it follows `.ai/operations.md` + RUNBOOK.

## Non-negotiable invariants
1. **Never touch the ACTIVE slot.** Build the idle slot; swap is an atomic symlink
   flip + `launchctl kickstart`. The `assert_not_active_slot` guard enforces this.
2. **Two human gates — always pause:** (a) before any **push to the fork**,
   (b) before the **live slot swap**. Everything else can be autonomous.
3. **Fresh parachute before deploy** (hermes-update does this automatically).
4. Work in supervised tmux shells (the `shared-terminals` skill), so Justin can watch.

## Branch model (see active-patches.yaml + versioned-patched-fork-strategy.md)
`origin/main == upstream/main` (mirror). `fixes/*`,`feat/*` are independent living
patches. `patched/vX` consolidates them. The consolidation is applied to the idle
color branch (`main-blue`/`main-green`) the slot tracks; then deploy + swap.
Old branches are preserved as `archive/*` tags.

**Every new feature MUST be registered** in `active-patches.yaml` (a `patches:`
entry with `source`, `commits:` oldest-first, `notes:`) *and* its branch pushed to
`origin`, or the resync drops it (the deploy branches are rebuilt from the patch
branches — anything only on `main-blue`/`main-green` is an orphan; a module can
survive while its call site + registration do not, which is how the routing
`oversight` wiring silently died). `hermes-manifest-check` enforces this as a
fail-closed `hermes-upgrade` preflight: dangling/unreachable manifest commits
abort; unpushed/unregistered branches warn. Full checklist: `.ai/operations.md`
→ "Authoring a new feature".

## Procedure

1. **Preflight:** `hermes-health` (must be green; note active vs idle slot). Confirm
   the dev repo `~/Dropbox/Dev/hermes-agent` is clean; `git fetch upstream origin`.
2. **Triage / plan:** read `~/.hermes/active-patches.yaml`. For each carried patch,
   check if upstream already has it (drop if merged). **Direct cherry-picks do NOT
   apply across large upstream gaps** — re-derive each patch from its ADR + its
   `archive/*` source against *current* upstream, not by replaying old diffs.
3. **Rebuild** clean single-concern branches on current `main`:
   - Additive modules (e.g. `agent/routing/`) copy clean; integration hooks re-wire
     against current upstream files. Verify imports (`python -c 'import ...'`).
   - **GATE:** push branches only after showing Justin the plan.
4. **Consolidate + deploy:** `hermes-upgrade` (in rescue/) — dry-run first, present
   the summary, then `--apply`. It builds `patched/vSTAMP`, updates the idle color
   branch, and calls `hermes-update` (gated: fresh rescue, active-slot guard, deps,
   tests, CA-bundle health, boot verify) to deploy the **idle** slot.
   - **GATE:** confirm with Justin before the swap. Green stays live until then.
5. **Verify + report:** `hermes-health`; summarize what changed, what to watch.

## Swap model (2026-08-07): slot-agnostic plist
The launchd plist references the `~/.hermes/hermes-agent` **symlink**, not a resolved
slot — so a swap is just `hermes-slot <color>` + `launchctl kickstart -k` (flip +
restart in place); the gateway follows the active slot with **no plist regen and no
wrong-slot risk**. `hermes-health` and `hermes-update` gate-8 resolve the symlink when
attributing the slot. ⚠ `hermes gateway install`/`--force` re-stamps the *resolved*
slot path (uses `sys.executable`) — if run, it reintroduces the wrong-slot footgun;
restore with `rescue/hermes-gateway-symlink-plist`. Gate 5 runs the test suite in a
throwaway `/tmp` git worktree so no test can corrupt the deploy slot (both baked into
`hermes-update`; no action needed).

## Recovery (always works, agent-independent)
- Dangling symlink / bad slot: `hermes-health --fix` (repoints to healthy twin), or
  the launchd wrapper `hermes-gateway-launch` fails over on its own.
- Manual swap: `hermes-slot green|blue|swap` then
  `launchctl kickstart -k gui/$(id -u)/ai.hermes.gateway`. After a manual swap
  (split-gate flow), run `hermes-ops-record` to commit the deploy-log + manifest
  (gate 9 does this automatically inside a full `hermes-update --apply`).
- Gateway on wrong slot (`hermes-health` CRIT STALE/mismatched, usually after a stray
  `hermes gateway install`): `rescue/hermes-gateway-symlink-plist` (restores the
  slot-agnostic plist + reloads).
- Total rebuild: `unzip` the newest `~/Dropbox/Dev/hermes-config-backup/rescue/hermes-rescue-*.zip`
  and `bash restore.sh`.

## Tooling (all in ~/Dropbox/Dev/hermes-config-backup/rescue/)
`hermes-upgrade` (this cycle), `hermes-update` (gated deploy; gate-5 tests run in a
`/tmp` worktree), `hermes-slot` (switch), `hermes-slot-guard` (read-only slots),
`hermes-gateway-symlink-plist` (restore slot-agnostic plist), `hermes-rescue`
(snapshot), `hermes-health[-cron]` (health/self-heal/alert; symlink-aware),
`hermes-gateway-launch` (failover launcher), `hermes-ops-record` (scoped commit of
`deploy-log.md` + `active-patches.yaml` to the LOCAL ops repo — no `-A`, no push;
auto-run by `hermes-update` gate 9, run standalone after a manual `hermes-slot` swap),
`hermes-manifest-check` (verify every patch is registered + reachable + pushed;
fail-closed `hermes-upgrade` preflight, run standalone anytime).
See memory `hermes-reliability-tooling`.
