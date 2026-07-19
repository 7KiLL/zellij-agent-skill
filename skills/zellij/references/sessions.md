# Session management and organization

Organizing the session **is** the skill: name what you create, put it on the
right surface (split, tab, floating), retag it as its state changes, move it
when the shape stops fitting, and tear down exactly what you created.

Everything on this page is plain `zellij action` — exact flag syntax lives
in [cli.md](cli.md). Running visible Claude/Codex agents over worktrees is
the same commands plus git: [agent-runs.md](agent-runs.md).

## Contents

- [Name everything at creation](#name-everything-at-creation)
- [Tagging — rename to reflect state](#tagging--rename-to-reflect-state)
- [Split, tab, or floating?](#split-tab-or-floating)
- [Moving things when the shape changes](#moving-things-when-the-shape-changes)
- [Know the state before acting](#know-the-state-before-acting)
- [Teardown](#teardown)

## Name everything at creation

An anonymous pane is unfindable ten minutes later. Always pass `--name` when
creating, and capture the printed pane id:

```sh
PANE=$(zellij action new-pane -d down --name "dev-server" -- npm run dev)
zellij action new-tab --name "feat-login" --cwd "$WT"
```

Name the **job**, not the tool: `dev-server`, `tests:watch`, `tail:api` — not
`bash` or `pane2`. The name is what the human scans; the id is what your
commands target.

## Tagging — rename to reflect state

A rename is one cheap command and instantly visible to the human — use it as
a status display:

```sh
zellij action rename-pane "ok:build"    --pane-id "$PANE"
zellij action rename-pane "fail:tests"  --pane-id "$PANE"
zellij action rename-tab  "review:auth"
```

Suggested prefixes: `run:` in progress, `ok:` finished clean, `fail:` needs
attention, `review:` waiting on the human. `undo-rename-pane` /
`undo-rename-tab` restore the automatic title when the tag no longer applies.

`rename-pane` targets `--pane-id`; `rename-tab` renames the focused tab
unless you pass `--tab-id` (ids from `list-tabs`).

## Split, tab, or floating?

| Surface | Use when | Create |
|---|---|---|
| Split pane | Output belongs *next to* current work: test watcher, log tail | `new-pane -d down\|right --name N -- CMD` |
| Tab | Separate concern with its own working directory — a worktree, a second service | `new-tab --name N --cwd DIR` |
| Floating | Glanceable and short-lived: one build, quick diff | `new-pane -f --name N -- CMD` |

Rules of thumb:

- More than ~3 splits in one tab is noise — promote the extra concern to a
  tab.
- One worktree = one tab is the natural mapping (see
  [SKILL.md](../SKILL.md#worktrees)).
- Floating panes hide with one `toggle-floating-panes` — right for things the
  user peeks at, wrong for things they must not miss.
- Need an exact shape (2×2 grid, N equal panes)? Declare it in a layout file
  and pass `new-tab --layout` —
  [agent-runs.md](agent-runs.md#grids-and-exact-layouts).

## Moving things when the shape changes

These are decisions, not projects — each is one command:

```sh
zellij action move-pane right --pane-id "$PANE"   # rearrange within the tab
zellij action toggle-pane-embed-or-floating       # float ⇄ tile the focused pane
zellij action stack-panes -- terminal_1 terminal_2 # collapse panes into a stack
zellij action move-tab right                      # reorder tabs
zellij action toggle-fullscreen                   # zoom the focused pane
```

Zellij 0.44 has no CLI action to move a pane into *another* tab. To relocate
a job: read its name/cwd/command from `list-panes -c -t`, close it, respawn
it in the target tab. For an idle shell pane, just close and recreate.

## Know the state before acting

Never act on a remembered id or name — look first:

```sh
zellij action list-panes -c -s -t --json   # panes: command, state, tab
zellij action list-tabs -p -s --json       # tabs: panes count, active state
zellij action query-tab-names              # just the names
zellij action current-tab-info             # where am I
```

Then read content with `dump-screen [--full] --pane-id ID` — details and the
reliable `tee` pattern are in
[SKILL.md](../SKILL.md#core-pattern-park-a-job-read-it-back).

## Teardown

Close what **you** created, and in the right order:

- Panes before worktrees — a pane cwd'd inside a worktree pins the directory.
- `close-pane --pane-id ID` — which is why you captured ids at creation.
- `close-tab` only hits the focused tab; close by name via
  `go-to-tab-name X && zellij action close-tab`, or use
  `close-tab-by-id` with an id from `list-tabs`.
- Never close panes/tabs you didn't create; the session belongs to the human.
