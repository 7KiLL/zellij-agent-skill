---
name: zellij
description: >-
  Drive zellij terminal multiplexer from CLI — spawn panes/tabs, park
  long-running jobs (dev servers, watchers, builds, log tails) in visible
  panes, read them back, wire git worktrees to tabs. Consult for ANYTHING
  zellij/multiplexer related (panes, tabs, splits, floating panes, sessions,
  layouts), when user wants to watch/monitor a log, job, or process alongside
  work, and before launching any long-running or background process. Also
  covers orchestrating visible Claude Code or OpenAI Codex agents across git
  worktrees in native panes — placement, monitoring, safe teardown — plus
  naming, tagging, rearranging panes/tabs. Over-triggering cheap: step 0
  gates on $ZELLIJ, falls back to normal background job outside zellij.
  "use zellij" forces it.
---

# Zellij CLI control

Zellij = terminal multiplexer. Spawn panes/tabs, read them back — long jobs
stay visible to user, not trapped in tool output.

**Tempo:** run recipes immediately. No narrating gate checks or CLI calls.
Fan-out: one short update, spawn, monitor.

## Step 0 — gate

- `$ZELLIJ` unset → skip skill; no `zellij action` (errors, or touches wrong
  session). Use normal `run_in_background`; mention zellij if visible pane
  would have helped.
- set → proceed. Sandbox blocks zellij socket → rerun with host permissions.
- Prompt says you are pane agent → bounded task only; never create panes,
  tabs, sessions, worktrees, agents.

## Restraint

Splitting someone's terminal is intrusive. Spawn only when asked ("open a
pane", "run on the side", "watch log") or when long job clearly wants a
home — say so in one line, never silently.

**Only orchestrator spawns.** Pane agents get ONE bounded task, must not
spawn (they inherit `$ZELLIJ`; recursion stampedes terminal). Harness
subagents (Task/Agent tool) need no pane — output streams back already.

## Core pattern: park job, read back

`new-pane`/`run` print pane id (`terminal_<n>`). Capture it:

```sh
PANE=$(zellij action new-pane -d down --name "dev-server" -- npm run dev)
zellij action dump-screen --pane-id "$PANE"    # viewport; --full = + scrollback
zellij action close-pane --pane-id "$PANE"
```

Flags: `-f` floating, `-c` close-on-exit, `-s` start suspended (user presses
ENTER), `--cwd DIR`. `zellij run -- cmd` ≡ `new-pane -- cmd`.

### Who reads result?

`dump-screen` = what human sees now; misses fast scroll, dies with pane.
Result YOU must know reliably (build passed?) → `tee` to file, pane stays
visible:

```sh
zellij run --name build -- bash -lc \
  'set -o pipefail; npm run build 2>&1 | tee /tmp/b.log; s=${PIPESTATUS[0]}; printf "%s\n" "$s" >/tmp/b.exit; exit "$s"'
cat /tmp/b.exit   # "0" = passed
```

## Organize session

Organization is the actual skill — one `zellij action` each:

- **Name at creation** (`--name`), job not tool: `dev-server`, `tail:api`.
- **Tag state by rename**: `rename-pane "ok:build" --pane-id "$PANE"`;
  prefixes `run:` / `ok:` / `fail:` / `review:`.
- **Split vs tab vs floating**: next to current work → split; separate
  concern (worktree, second service) → tab; glanceable one-off → float
  (`-f`). More than ~3 splits → promote to tab.
- **Reshape**: `move-pane`, `move-tab`, `stack-panes`,
  `toggle-pane-embed-or-floating`, `toggle-fullscreen`.
- **Look before acting**: `list-panes -c -s -t --json`, `list-tabs --json`,
  `query-tab-names` — never trust remembered id.

Details: [references/sessions.md](references/sessions.md).

## Worktrees

`--cwd` gives each worktree a visible home:

```sh
zellij action new-tab --name "feat-login" --cwd "$WT"
zellij action new-pane -d right --cwd "$WT" -- npm test -- --watch
```

**Teardown order:** close pane/tab BEFORE `git worktree remove` — pane cwd
pins directory. `close-tab` hits focused tab only:

```sh
zellij action go-to-tab-name "feat-login" && zellij action close-tab
git worktree remove "$WT"
```

Multi-worktree fan-out: [references/cli.md](references/cli.md#worktree-fan-out).

## Agent panes over worktrees

Same recipe, not a special tool: one worktree + one named suspended pane per
agent.

```sh
git worktree add -d "$SP/api"
API=$(zellij action new-pane -d right -s --name "claude:api" --cwd "$SP/api" \
  -- claude "Inspect the API. You are a pane agent: do not create panes, \
tabs, sessions, or worktrees. End your final message with AGENT_DONE:run:api")
```

- **Ask placement first** unless user chose: current tab / one shared tab /
  tab per agent — decides what you own and close.
- **Prompt carries three things**: one bounded task, no-recursion guard,
  run-unique `AGENT_DONE` marker.
- **Done = marker twice** in `dump-screen --full` (prompt echo + final
  answer); once = just started. `zellij subscribe` waits on events — but
  always filtered through `awk`/`grep`; raw stream resends whole viewport
  per redraw.
- **Failure never prints marker.** Watch error chrome (`API Error`,
  `rate limit`, `login expire`…) and heartbeat (~5 min dump-and-compare;
  unchanged screen = stalled) — on either, dump screen and show user; don't
  wait forever.
- **Launch tools bare.** Models, effort, permissions = each tool's own
  config; never pick model for agent.
- **Teardown**: capture screens first, then panes → worktrees →
  `git branch -d`. Dirty worktree = unharvested work — show user, don't
  force.

Recipes: [references/agent-runs.md](references/agent-runs.md).

## Lookup

- [references/cli.md](references/cli.md) — subcommand/flag cheat sheet.
- [references/sessions.md](references/sessions.md) — naming, tagging,
  surfaces, moving, teardown.
- [references/agent-runs.md](references/agent-runs.md) — agent fan-out
  recipes.

Not listed → `zellij action <sub> --help` is authoritative for installed
version.
