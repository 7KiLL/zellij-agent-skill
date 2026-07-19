# Agent fan-out with plain zellij

Spawn visible Claude Code or Codex agents in native panes, one Git worktree
each — no controller, no binary, just `zellij action` and `git`. Command
syntax: [cli.md](cli.md). Organization rules: [sessions.md](sessions.md).

## Contents

- [The pattern](#the-pattern)
- [Placement is a real question](#placement-is-a-real-question)
- [Grids and exact layouts](#grids-and-exact-layouts)
- [Prompting a pane agent](#prompting-a-pane-agent)
- [Monitoring](#monitoring)
- [Teardown](#teardown)
- [Rules](#rules)

## The pattern

One worktree, one named pane, one suspended agent per task:

```sh
SP=/abs/worktrees; RUN=audit
git worktree add -d "$SP/api"
API=$(zellij action new-pane -d right -s --name "claude:api" --cwd "$SP/api" \
  -- claude "Inspect the API. You are a pane agent: do not create panes, \
tabs, sessions, or worktrees. End your final message with AGENT_DONE:$RUN:api")

git worktree add -d "$SP/tests"
TESTS=$(zellij action new-pane -d down -s --name "codex:tests" --cwd "$SP/tests" \
  -- codex "Inspect the tests. You are a pane agent: do not create panes, \
tabs, sessions, or worktrees. End your final message with AGENT_DONE:$RUN:tests")
```

`-s` leaves each pane at Zellij's native Enter-to-run screen — the user
starts each agent deliberately. Drop `-s` to start immediately. Capture the
printed pane ids; they are your only handle for monitoring and teardown.

Launch each tool bare: `claude "task"` / `codex "task"`. Models, effort,
permission modes, and sandbox flags are each tool's own business — the user
configures their tools; never pick a model or bolt one tool's flags onto the
other.

## Placement is a real question

Ask where panes should go unless the user already chose — it decides what
you own and must close later:

| Choice | Create | You own |
|---|---|---|
| Current tab | `new-pane -d right\|down` beside you | Only the agent panes |
| One shared tab | `new-tab --name "$RUN"` first, then panes in it | That tab |
| Separate tabs | `new-tab --name "$RUN:api" --cwd "$WT"` per agent | Each created tab |

**`new-pane` has no tab target — it inserts into the session's *focused*
tab, and focus is shared with the human.** `new-tab` focuses the new tab and
an immediate `new-pane` in the same script lands correctly, but the moment
your commands are split across tool calls, a human tab-click in between
redirects every later spawn (verified: panes follow whatever tab is active
at spawn time). So: create the tab and spawn **all** panes in one script, or
re-assert `go-to-tab-name "$RUN"` immediately before each `new-pane` — then
confirm with `list-panes -t` before relying on the layout.

A bare `new-tab` also always contains **one default shell pane** (a tab
cannot be empty) — close it after spawning your agent panes, or avoid it
entirely with a layout file below.

## Grids and exact layouts

`new-tab --layout FILE` creates a tab with *exactly* the panes you declare —
no stray shell pane, precise geometry. A 2×2 agent grid (verified: four
equal quadrants, bars kept):

```kdl
// grid.kdl — zellij action new-tab --name "$RUN" --layout grid.kdl
layout {
    pane size=1 borderless=true { plugin location="zellij:tab-bar"; }
    pane split_direction="vertical" {
        pane split_direction="horizontal" {
            pane name="claude:api"  command="claude" start_suspended=true cwd="/abs/wt/api"  { args "task…"; }
            pane name="claude:ui"   command="claude" start_suspended=true cwd="/abs/wt/ui"   { args "task…"; }
        }
        pane split_direction="horizontal" {
            pane name="codex:tests" command="codex" start_suspended=true cwd="/abs/wt/tests" { args "task…"; }
            pane name="codex:docs"  command="codex" start_suspended=true cwd="/abs/wt/docs"  { args "task…"; }
        }
    }
    pane size=1 borderless=true { plugin location="zellij:status-bar"; }
}
```

`split_direction="vertical"` = side-by-side columns, `"horizontal"` =
stacked rows; nest them for any grid. Keep the two bar panes or the tab
loses its tab/status bars. A layout prints no pane ids — recover them by
the `name=` values via `list-panes -t -j`. Prompts containing quotes are
painful to escape in KDL; for those, fall back to sequential `new-pane`
(normal shell quoting) and close the default pane.

## Prompting a pane agent

Every pane-agent prompt carries three parts:

1. **One bounded task.** A pane agent gets a single deliverable, not a
   conversation.
2. **The no-recursion guard.** Pane agents inherit `$ZELLIJ`; without the
   "do not create panes/tabs/sessions/worktrees" line they may stampede the
   terminal.
3. **A done marker** unique to run and agent (`AGENT_DONE:RUN:AGENT`), so
   completion is detectable from the screen.

## Monitoring

```sh
zellij action dump-screen --full --pane-id "$API" --path /tmp/api.txt
grep -c "AGENT_DONE:$RUN:api" /tmp/api.txt   # ≥2 = done (prompt echo + final answer)
```

The marker appears once in the echoed prompt from the start — require it
**twice** before treating the agent as finished, or a fresh pane reads as
done.

**Never read the raw `zellij subscribe` stream** — every `pane_update` event
re-sends the full rendered viewport, and a busy TUI redraws constantly
(measured: ~94 KB per 5 s for one active pane; an idle pane costs ~nothing).
Pipe it through a filter that stays silent until the condition, so hours of
waiting cost one output line:

```sh
zellij subscribe --pane-id "$API" --format json \
  | awk '/"event":"pane_closed"/{print "closed"; exit}
         {n=gsub(/AGENT_DONE:run:api/,"&")} n>=2{print "done"; exit}'
```

Same token cost, lower tech: poll `dump-screen --full … | grep -c` every
minute or two — one viewport per check. A pane needing input (auth,
permission prompt) just sits there — `dump-screen` it and tell the user
rather than waiting forever.

The marker is the signal to **act**, not proof the agent stopped: an agent
in auto mode may queue itself follow-up work past its marker (observed
live). On marker detection, capture the screen and close the pane promptly —
don't wait for it to go idle.

Before closing anything, save what matters: dump each screen to a file and
note each worktree's branch tip — panes and scrollback die with teardown.

## Teardown

Order is load-bearing — panes pin their cwd:

```sh
zellij action close-pane --pane-id "$API"
zellij action close-pane --pane-id "$TESTS"
git worktree remove "$SP/api"        # refuses if dirty — good; never rm -rf
git worktree remove "$SP/tests"
git branch -d <branch>               # only with -d: unmerged work survives
```

A dirty worktree means unharvested work: show the user, don't `--force`. If
you created a shared/separate tab, close it by name
(`go-to-tab-name X && zellij action close-tab`); never close the user's own
tabs.

## Rules

- Only the orchestrator spawns and monitors panes; agents get one task each.
- A dirty source checkout at spawn time is worth a warning — worktrees carry
  committed state only. Never auto-commit the user's checkout to fix that.
- Wrong tool flags are worse than no flags: launch tools bare and let user
  config decide.
