---
name: zellij
description: >-
  Drive the zellij terminal multiplexer from the CLI — spawn panes and tabs,
  park long-running jobs (dev servers, watchers, builds, log tails) in a
  visible pane and read them back, wire git worktrees to tabs. Consult for
  ANYTHING zellij- or multiplexer-related (panes, tabs, splits, floating
  panes, sessions, layouts), whenever the user wants to watch or monitor a
  log, job, or process alongside other work, and before launching any
  long-running or background process. Also covers orchestrating visible
  Claude Code or OpenAI Codex agents across git worktrees through native
  Zellij panes in the current tab, one shared tab, or separate tabs, with
  monitoring and safe teardown. Covers naming, tagging, and rearranging
  panes and tabs to keep a session organized.
  Over-triggering is cheap: step 0 gates on $ZELLIJ and falls back to a normal
  background job when not inside zellij — so consult even when unsure. Saying
  "use zellij" forces it.
---

# Zellij CLI control

Zellij is a terminal multiplexer. Inside it you can spawn panes/tabs and read
them back — keep a long job visible to the user instead of trapped in your
tool output.

**Tempo:** run mechanical recipes immediately. Do not explain that Zellij was
detected, recite the gate, or narrate each CLI call. For an agent fan-out,
write one short update, spawn the panes, then monitor them.

## Step 0 — gate

Use `$ZELLIJ` as the first gate:

- **unset** → skip this skill; don't run `zellij action` (it errors or, worse,
  touches another session). Fall back to a normal `run_in_background` job and,
  if a visible pane would have helped, mention zellij would provide that.
- **set** → proceed. If sandbox isolation blocks the zellij socket, rerun the
  same command with host/escalated permissions.
- **your prompt says you are a pane agent** → perform the bounded task only;
  never create panes, tabs, sessions, worktrees, or more agents.

## Restraint

Splitting someone's terminal is intrusive. Spawn panes/tabs only when asked
("open a pane", "run it on the side", "watch the log") or when a long job
clearly wants a home — and say so in one line as you do it, never silently.

**Only the orchestrator spawns.** You (the top-level agent) spawn every pane
and monitor every pane. Pane agents get ONE bounded task and must not spawn
more panes themselves (they inherit `$ZELLIJ`, and recursive spawning
stampedes the terminal). Harness subagents (Task/Agent tool) need no pane —
their output already streams back.

## Core pattern: park a job, read it back

`new-pane`/`run` print the created pane id (`terminal_<n>`). Capture it:

```sh
PANE=$(zellij action new-pane -d down --name "dev-server" -- npm run dev)
zellij action dump-screen --pane-id "$PANE"    # viewport; --full = + scrollback
zellij action close-pane --pane-id "$PANE"     # cleanup when done
```

Flags: `-f` floating, `-c` close-on-exit, `-s` start suspended (runs when the
user presses ENTER), `--cwd DIR`. `zellij run -- cmd` ≡ `new-pane -- cmd`.

### Who reads the result?

`dump-screen` shows what the human sees now — it can miss fast-scrolling
output and dies with the pane. For a result YOU must know reliably (did the
build pass?), use `tee` so the pane stays useful while a file records it:

```sh
zellij run --name build -- bash -lc \
  'set -o pipefail; npm run build 2>&1 | tee /tmp/b.log; s=${PIPESTATUS[0]}; printf "%s\n" "$s" >/tmp/b.exit; exit "$s"'
cat /tmp/b.exit   # "0" = passed — pane stays visible for the human
```

## Organize the session

Organization is the actual skill, and it needs no controller — one
`zellij action` each:

- **Name at creation** (`--name`), name the job not the tool: `dev-server`,
  `tail:api`.
- **Tag state by renaming**: `rename-pane "ok:build" --pane-id "$PANE"` —
  prefixes like `run:` / `ok:` / `fail:` / `review:` make a pane a status
  display.
- **Split vs tab vs floating**: output that belongs next to current work
  splits; a separate concern (worktree, second service) gets a tab; a
  glanceable one-off floats (`-f`). More than ~3 splits → promote to a tab.
- **Reshape freely**: `move-pane`, `move-tab`, `stack-panes`,
  `toggle-pane-embed-or-floating`, `toggle-fullscreen`.
- **Look before acting**: `list-panes -c -s -t --json`, `list-tabs --json`,
  `query-tab-names` — never trust a remembered id.

Patterns and decision rules: [references/sessions.md](references/sessions.md).

## Worktrees

`--cwd` gives each worktree a visible home:

```sh
zellij action new-tab --name "feat-login" --cwd "$WT"
zellij action new-pane -d right --cwd "$WT" -- npm test -- --watch
```

**Teardown order:** close the pane/tab *before* `git worktree remove` — a pane
cwd'd inside the worktree pins the directory. `close-tab` only hits the
focused tab, so close by name via:

```sh
zellij action go-to-tab-name "feat-login" && zellij action close-tab
git worktree remove "$WT"
```

Multi-worktree fan-out (spawn N panes, tear all down): see
[references/cli.md](references/cli.md#worktree-fan-out).

## Agent panes over worktrees

Visible Claude Code / Codex agents are the same recipe, not a special tool:
one worktree + one named, suspended pane per agent.

```sh
git worktree add -d "$SP/api"
API=$(zellij action new-pane -d right -s --name "claude:api" --cwd "$SP/api" \
  -- claude "Inspect the API. You are a pane agent: do not create panes, \
tabs, sessions, or worktrees. End your final message with AGENT_DONE:run:api")
```

- **Ask placement first** unless the user chose: panes in the current tab,
  one shared tab, or a tab per agent — it decides what you own and close.
- **The prompt carries three things**: one bounded task, the no-recursion
  guard, and a run-unique `AGENT_DONE` marker.
- **Done = marker twice** in `dump-screen --full` output (prompt echo + final
  answer); once means it just started. `zellij subscribe` waits on events
  instead of polling — but always filtered through `awk`/`grep`; the raw
  stream re-sends the whole viewport on every redraw.
- **Launch tools bare.** Models, effort, permissions are each tool's own
  business — the user's config decides; never pick a model for an agent.
- **Teardown**: capture screens first, then panes → worktrees →
  `git branch -d`. A dirty worktree is unharvested work — show the user,
  don't force.

Full recipes, monitoring, and teardown rules:
[references/agent-runs.md](references/agent-runs.md).

## Command lookup

- [references/cli.md](references/cli.md) — full subcommand/flag cheat sheet.
- [references/sessions.md](references/sessions.md) — organization patterns:
  naming, tagging, surface choice, moving, teardown.
- [references/agent-runs.md](references/agent-runs.md) — agent fan-out
  recipes.

For anything not listed, `zellij action <sub> --help` is authoritative for
the installed version — check it rather than guessing.
