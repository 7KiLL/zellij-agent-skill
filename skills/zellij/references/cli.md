# zellij CLI reference (0.44)

All commands `zellij action <sub>` unless noted. Directions:
`right|left|up|down` (`new-pane` splits only `right|down`). Patterns:
[sessions.md](sessions.md). Agent fan-out: [agent-runs.md](agent-runs.md).

## Panes

- `new-pane [-d down|right] [-f] [--name N] [--cwd DIR] [-c] [-s] [-- CMD]` — prints id (`terminal_<n>`)
- `close-pane [--pane-id ID]`
- `move-focus <dir>` / `move-focus-or-tab <dir>` — latter crosses tab edges
- `focus-next-pane` / `focus-previous-pane` / `focus-pane-id <id>`
- `rename-pane <name> [--pane-id ID]` / `undo-rename-pane`
- `move-pane [dir] [--pane-id ID]` — within its tab
- `stack-panes -- <id id...>` — collapse into stacked group
- `toggle-pane-embed-or-floating` — float ⇄ tile focused pane
- `resize <dir|+|->`
- `toggle-fullscreen` — zoom focused pane
- `toggle-floating-panes` — show/hide floating layer
- `paste [--pane-id ID] <str>` — bracketed paste
- `send-keys [--pane-id ID] <keys...>` — named keys e.g. `Enter`
- `write-chars <str>` / `write <bytes>` — type into focused pane
- `clear` — clear focused pane buffers
- `dump-screen [--full] [--pane-id ID] [--path FILE]` — viewport (stdout default; `--full` adds scrollback)
- `list-panes [-a] [-c] [-s] [-t] [-g] [--json]` — ids, commands, state, tab, geometry

`zellij subscribe --pane-id ID... [--format raw|json] [--scrollback [N]]` —
streams `pane_update`/`pane_closed`, ends when watched panes/session close;
one subscription waits on many panes. Each `pane_update` = full viewport,
active pane ~1 MB/min: ALWAYS pipe through grep/awk
([agent-runs.md](agent-runs.md#monitoring)), never into own output. Capture
and final confirmation: `dump-screen --full`. Older builds lack subscribe —
fall back to bounded polling.

Floating geometry (with `-f`): `-x/-y/--width/--height` int or percent
(`10%`); `--pinned true` keeps on top.

## Tabs

- `new-tab [--name N] [--cwd DIR] [--layout PATH] [-- CMD]` — prints position. Without `--layout` always starts with one default shell pane — close it after spawning, or define exact panes via layout: [agent-runs.md](agent-runs.md#grids-and-exact-layouts)
- `close-tab` — focused tab only (by name: `go-to-tab-name X && close-tab`); `close-tab-by-id <id>`
- `go-to-tab <index>` / `go-to-tab-name <name> [--create]` / `go-to-tab-by-id <id>` / `go-to-next-tab` / `go-to-previous-tab`
- `rename-tab <name> [--tab-id ID]` / `undo-rename-tab`
- `move-tab <right|left> [--tab-id ID]`
- `query-tab-names`
- `list-tabs [-a] [-p] [-s] [--json]` — ids, pane counts, state
- `current-tab-info`

## Session / files

- `zellij list-sessions`
- `zellij action edit <file> [-f] [-i]` — open file in $EDITOR pane
- `zellij action edit-scrollback`
- `zellij action detach` — leaves everything running

## Worktree fan-out

Capture pane ids at creation — `close-pane` takes `--pane-id`, not `--name`;
can't recover later:

```sh
SP=/abs/parent-dir; NAMES="wt1 wt2"; IDS=""
for w in $NAMES; do
  git worktree add -d "$SP/$w"
  IDS="$IDS $(zellij action new-pane -d right --name "$w" --cwd "$SP/$w" \
    -- sh -c 'git status -sb; echo; ls; exec $SHELL')"
done
# teardown — panes first (pin cwd), then worktrees
for id in $IDS; do zellij action close-pane --pane-id "$id"; done
for w  in $NAMES; do git worktree remove "$SP/$w"; done
```

Visible agent per worktree pane: [agent-runs.md](agent-runs.md).
