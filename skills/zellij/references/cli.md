# zellij CLI reference (0.44)

All commands are `zellij action <sub>` unless noted. Direction args:
`right|left|up|down` (`new-pane` only splits `right|down`). Organization
patterns built on these commands: [sessions.md](sessions.md). Agent fan-out
recipes: [agent-runs.md](agent-runs.md).

## Contents

- [Panes](#panes)
- [Tabs](#tabs)
- [Session and files](#session--files)
- [Worktree fan-out](#worktree-fan-out)

## Panes

- `new-pane [-d down|right] [-f] [--name N] [--cwd DIR] [-c] [-s] [-- CMD]` — new pane; prints id (`terminal_<n>`)
- `close-pane [--pane-id ID]` — close focused (or given) pane
- `move-focus <dir>` / `move-focus-or-tab <dir>` — move focus; latter crosses tab edges
- `focus-next-pane` / `focus-previous-pane` / `focus-pane-id <id>`
- `rename-pane <name> [--pane-id ID]` / `undo-rename-pane` — set/clear pane title
- `move-pane [dir] [--pane-id ID]` — relocate a pane within its tab
- `stack-panes -- <id id...>` — collapse panes into one stacked group
- `toggle-pane-embed-or-floating` — float ⇄ tile the focused pane
- `resize <dir|+|->` — grow/shrink focused pane
- `toggle-fullscreen` — zoom focused pane
- `toggle-floating-panes` — show/hide the floating layer
- `paste [--pane-id ID] <str>` — bracketed-paste text into a pane
- `send-keys [--pane-id ID] <keys...>` — send named keys such as `Enter`
- `write-chars <str>` / `write <bytes>` — type into the focused pane
- `clear` — clear focused pane's buffers
- `dump-screen [--full] [--pane-id ID] [--path FILE]` — viewport (stdout by default; `--full` adds scrollback)
- `list-panes [-a] [-c] [-s] [-t] [-g] [--json]` — pane ids, commands, state, tab, geometry

`zellij subscribe --pane-id ID... [--format raw|json] [--scrollback [N]]`
streams rendered `pane_update` and `pane_closed` events and ends when the
watched panes or session close — use one subscription to wait on many panes
instead of polling. Each `pane_update` carries the **full viewport**, so an
active pane emits ~1 MB/min: always pipe through `grep`/`awk`
([agent-runs.md](agent-runs.md#monitoring)), never into your own output.
Use `dump-screen --full` for capture and final-marker confirmation. Older
Zellij builds lack it — fall back to bounded polling.

Floating-pane geometry (with `-f`): `-x/-y/--width/--height` as integer or
percent (e.g. `10%`); `--pinned true` keeps it always on top.

## Tabs

- `new-tab [--name N] [--cwd DIR] [--layout PATH] [-- CMD]` — new tab
- `close-tab` — close the *focused* tab (close by name: `go-to-tab-name X && close-tab`); `close-tab-by-id <id>`
- `go-to-tab <index>` / `go-to-tab-name <name> [--create]` / `go-to-tab-by-id <id>` / `go-to-next-tab` / `go-to-previous-tab`
- `rename-tab <name> [--tab-id ID]` / `undo-rename-tab` — set/clear tab title
- `move-tab <right|left> [--tab-id ID]` — reorder tabs
- `query-tab-names` — list tab names (check state before acting)
- `list-tabs [-a] [-p] [-s] [--json]` — tab ids, pane counts, state
- `current-tab-info` — details on the active tab

## Session / files

- `zellij list-sessions` — active sessions
- `zellij action edit <file> [-f] [-i]` — open file in a new pane in $EDITOR
- `zellij action edit-scrollback` — open focused pane's scrollback in $EDITOR
- `zellij action detach` — detach, leaves everything running

## Worktree fan-out

Spawn N worktrees each in a named pane, then tear down. Capture pane ids at
creation — `close-pane` takes `--pane-id`, not `--name`, so you can't recover
them later:

```sh
SP=/abs/parent-dir; NAMES="wt1 wt2"; IDS=""
for w in $NAMES; do
  git worktree add -d "$SP/$w"
  IDS="$IDS $(zellij action new-pane -d right --name "$w" --cwd "$SP/$w" \
    -- sh -c 'git status -sb; echo; ls; exec $SHELL')"
done
# teardown — panes first (they pin the cwd), then the worktrees
for id in $IDS; do zellij action close-pane --pane-id "$id"; done
for w  in $NAMES; do git worktree remove "$SP/$w"; done
```

To run a visible Claude/Codex agent in each worktree pane instead of a
shell, see [agent-runs.md](agent-runs.md).
