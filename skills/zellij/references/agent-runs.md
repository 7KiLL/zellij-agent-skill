# Agent fan-out with plain zellij

Visible Claude Code / Codex agents in native panes, one git worktree each —
no controller, no binary, just `zellij action` + `git`. Syntax:
[cli.md](cli.md). Organization: [sessions.md](sessions.md).

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

`-s` = Zellij's Enter-to-run screen; user starts each agent deliberately.
Drop `-s` to start immediately. Capture printed pane ids — only handle for
monitoring and teardown.

Single ad-hoc agent: Claude's own worktree flag, skip manual
`git worktree add`:

```sh
zellij run --name scratch -- claude -w fix-auth "One bounded task."
```

Creates `.claude/worktrees/fix-auth` (branch `worktree-fix-auth`),
keep/remove prompt at exit. NOT for fan-out: branches from default branch
(origin/HEAD) not current HEAD — agents silently miss local work; tool picks
path/branch names; exit prompt races marker teardown; Codex has no
equivalent.

Launch tools bare: `claude "task"` / `codex "task"`. Models, effort,
permissions, sandbox = each tool's own config; never pick model or bolt one
tool's flags onto other.

## Placement

Ask where panes go unless user chose — decides what you own and close:

| Choice | Create | You own |
|---|---|---|
| Current tab | `new-pane -d right\|down` beside you | Agent panes only |
| One shared tab | `new-tab --name "$RUN"`, then panes in it | That tab |
| Separate tabs | `new-tab --name "$RUN:api" --cwd "$WT"` each | Each tab |

**`new-pane` has NO tab target — inserts into session's FOCUSED tab; focus
shared with human.** `new-tab` + immediate `new-pane` in same script lands
correctly, but commands split across tool calls = human tab-click between
them redirects later spawns (verified: panes follow active tab at spawn
time). So: tab + ALL panes in one script, or `go-to-tab-name "$RUN"`
immediately before each `new-pane`; confirm with `list-panes -t`.

Bare `new-tab` always contains one default shell pane (tab can't be empty)
— close it after spawning, or avoid via layout file below.

## Grids and exact layouts

`new-tab --layout FILE` = exactly the declared panes — no stray shell,
precise geometry. 2×2 grid (verified: four equal quadrants, bars kept):

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

`vertical` = side-by-side columns, `horizontal` = stacked rows; nest for any
grid. Keep the two bar panes or tab loses its bars. Layout prints no pane
ids — recover by `name=` via `list-panes -t -j`. Quotes in prompts = painful
KDL escaping — fall back to sequential `new-pane` (shell quoting) + close
default pane.

## Prompting a pane agent

Prompt carries three parts:

1. **One bounded task** — single deliverable, not a conversation.
2. **No-recursion guard** — agents inherit `$ZELLIJ`; without "do not create
   panes/tabs/sessions/worktrees" they may stampede terminal.
3. **Done marker** unique per run+agent (`AGENT_DONE:RUN:AGENT`).

## Monitoring

```sh
zellij action dump-screen --full --pane-id "$API" --path /tmp/api.txt
grep -c "AGENT_DONE:$RUN:api" /tmp/api.txt   # ≥2 = done (echo + answer)
```

Marker appears once in echoed prompt from start — require it TWICE or a
fresh pane reads done.

**Listen for trouble, not just marker.** Failed agent never prints it — API
error, rate limit, expired login, permission prompt all look exactly like
"still working". Two signals:

- **Error chrome** — tool failure strings: `API Error`, `Retrying in`,
  `rate limit`, `usage limit`, `stream error`, `login expire`. Match = LOOK
  NOW, not failed: dump screen, decide — auto-retry in progress = keep
  waiting; hard failure/auth = show user. False trips cheap (agent merely
  discussing rate limits costs one glance).
- **Heartbeat** — every ~5 min dump screen, compare previous. Changed =
  alive; identical + no marker = stalled (input wait, or tool died).
  Catches what string list misses.

  ```sh
  zellij action dump-screen --full --pane-id "$API" --path /tmp/api.txt
  cmp -s /tmp/api.txt /tmp/api.prev && echo "STALLED"
  mv -f /tmp/api.txt /tmp/api.prev
  ```

Trouble/stall: capture screen, tell user. Never auto-retry, never close
before capturing, never wait silently.

**Never read raw `zellij subscribe` stream** — each `pane_update` resends
full viewport; busy TUI redraws constantly (measured ~94 KB per 5 s active
pane; idle ~nothing). Filter, silent until condition — done, closed, or
trouble:

```sh
zellij subscribe --pane-id "$API" --format json \
  | awk '/"event":"pane_closed"/{print "closed"; exit}
         {n=gsub(/AGENT_DONE:run:api/,"&")} n>=2{print "done"; exit}
         tolower($0) ~ /api error|retrying in|rate limit|usage limit|stream error|login expire/{print "trouble"; exit}'
```

Done check BEFORE trouble check: finished agent with stale `Retrying in…`
on screen reads done. Stall emits no event, matches no pattern — subscribe
can't see it; keep heartbeat alongside.

Lower tech, same cost: poll `dump-screen --full … | grep -c` every minute or
two — heartbeat comes free (previous dump already in hand to `cmp`).

Marker = signal to ACT, not proof agent stopped: auto mode may queue
follow-up work past marker (observed live). On marker: capture screen, close
pane promptly.

Before closing anything: dump each screen to file, note each worktree's
branch tip — panes and scrollback die with teardown.

## Teardown

Order load-bearing — panes pin cwd:

```sh
zellij action close-pane --pane-id "$API"
zellij action close-pane --pane-id "$TESTS"
git worktree remove "$SP/api"        # refuses if dirty — good; never rm -rf
git worktree remove "$SP/tests"
git branch -d <branch>               # -d only: unmerged work survives
```

Dirty worktree = unharvested work: show user, don't `--force`. Created a
shared/separate tab → close by name
(`go-to-tab-name X && zellij action close-tab`); never close user's own
tabs.

## Rules

- Only orchestrator spawns/monitors; agents get one task each.
- Dirty source checkout at spawn = warn — worktrees carry committed state
  only. Never auto-commit user's checkout to fix.
- Wrong tool flags worse than none: launch bare, user config decides.
