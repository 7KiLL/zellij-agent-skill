# Session management and organization

Organizing the session IS the skill: name what you create, put it on right
surface (split/tab/floating), retag as state changes, move when shape stops
fitting, tear down exactly what you created. Flag syntax: [cli.md](cli.md).
Agents over worktrees: [agent-runs.md](agent-runs.md).

## Name everything at creation

Anonymous pane = unfindable ten minutes later. Always `--name`, capture
printed id:

```sh
PANE=$(zellij action new-pane -d down --name "dev-server" -- npm run dev)
zellij action new-tab --name "feat-login" --cwd "$WT"
```

Name the JOB, not tool: `dev-server`, `tests:watch`, `tail:api` — not `bash`
or `pane2`. Name = what human scans; id = what commands target.

## Tagging — rename to reflect state

Rename = one cheap command, instantly visible — use as status display:

```sh
zellij action rename-pane "ok:build"    --pane-id "$PANE"
zellij action rename-pane "fail:tests"  --pane-id "$PANE"
zellij action rename-tab  "review:auth"
```

Prefixes: `run:` in progress, `ok:` clean, `fail:` needs attention,
`review:` waiting on human. `undo-rename-pane` / `undo-rename-tab` restore
auto titles.

`rename-pane` targets `--pane-id`; `rename-tab` hits focused tab unless
`--tab-id` (ids from `list-tabs`).

## Split, tab, or floating?

| Surface | When | Create |
|---|---|---|
| Split | Output belongs next to current work: test watcher, log tail | `new-pane -d down\|right --name N -- CMD` |
| Tab | Separate concern, own cwd: worktree, second service | `new-tab --name N --cwd DIR` |
| Floating | Glanceable, short-lived: one build, quick diff | `new-pane -f --name N -- CMD` |

- More than ~3 splits in one tab = noise — promote to tab.
- One worktree = one tab ([SKILL.md](../SKILL.md#worktrees)).
- Floating hides with one `toggle-floating-panes` — right for peeking, wrong
  for must-not-miss.
- Exact shape (2×2 grid, N equal panes) → layout file + `new-tab --layout`:
  [agent-runs.md](agent-runs.md#grids-and-exact-layouts).

## Moving — one command each

```sh
zellij action move-pane right --pane-id "$PANE"   # rearrange within tab
zellij action toggle-pane-embed-or-floating       # float ⇄ tile
zellij action stack-panes -- terminal_1 terminal_2
zellij action move-tab right
zellij action toggle-fullscreen                   # zoom
```

0.44 has NO cross-tab pane move. Relocate: read name/cwd/command via
`list-panes -c -t`, close, respawn in target tab.

## Know state before acting

Never act on remembered id/name:

```sh
zellij action list-panes -c -s -t --json   # panes: command, state, tab
zellij action list-tabs -p -s --json
zellij action query-tab-names
zellij action current-tab-info
```

Read content: `dump-screen [--full] --pane-id ID`; reliable `tee` pattern:
[SKILL.md](../SKILL.md#core-pattern-park-job-read-back).

## Teardown

- Panes before worktrees — pane cwd pins directory.
- `close-pane --pane-id ID` — why you captured ids.
- `close-tab` hits focused tab; by name:
  `go-to-tab-name X && zellij action close-tab`, or `close-tab-by-id`.
- Never close panes/tabs you didn't create; session belongs to human.
