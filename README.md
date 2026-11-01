# idemux

A barebones IDE built on tmux.

One tmux session, one window per git repository, two panes per window:

```
┌────────────────────────────────┬────────────────────┐
│                                │                    │
│  editor  <->  lazygit           │  agent  <->  shell │
│                                │                    │
└────────────────────────────────┴────────────────────┘
              prefix+g                  prefix+a
```

The right pane starts at one third of the window width; the left pane takes the
remaining two thirds. Every pane starts in the repository root. The hidden side
of each toggle keeps running in a parking session, so switching between nvim and
lazygit (or between the agent and a shell) does not lose state.

## Requirements

- Linux (the repo scan uses GNU `find -printf`)
- tmux >= 3.2
- `git` and `fzf`
- an editor, `lazygit`, and a coding-agent CLI
- bash 4.4+ (arrays, `mapfile -d`)

## Install

Keep the script in the repository; it finds `idemux.conf` relative to itself.

```sh
git clone <this-repo> ~/gh/idemux
ln -s ~/gh/idemux/bin/idemux ~/.local/bin/idemux
```

Make sure `~/.local/bin` is on your `PATH`.

## First run

```sh
idemux
```

1. If `~/.config/idemux/config` does not exist, a small wizard asks for:
   - one or more directories to search for git repositories
   - the editor command (default `nvim`)
   - the coding-agent command (e.g. `claude`, `aider`, `codex`)
2. It scans those roots and opens an fzf picker. `TAB` multi-selects, `ENTER`
   opens one window per selected repository.
3. It creates a **dedicated** tmux server (`tmux -L idemux`) and attaches.

On later runs `idemux` attaches to the existing session without showing the picker.
If the IDE session is gone (for example, you closed the last repo), the picker
appears again.

## Keys

The dedicated server uses Ctrl+backtick as its prefix. The blue status bar sits
at the top and shows these bindings on the right as a cheat sheet.

| Key          | Action                                      |
|--------------|---------------------------------------------|
| `prefix+Tab` | move focus to the other pane                |
| `prefix+g`   | left pane: editor <-> lazygit               |
| `prefix+a`   | right pane: agent <-> shell                 |
| `prefix+o`   | open more repos with fzf                    |
| `prefix+X`   | close the current repo (with confirmation)  |
| `prefix+x`   | same as above (case-insensitive shortcut)   |
| `prefix+r`   | reload `idemux.conf` and clean up panes     |

Both `prefix+x` and `prefix+X` close the whole repo window and its parked
panes; this replaces tmux's default `kill-pane` binding on `x`. Use
`prefix+:kill-pane` if you really need to kill a single pane.

> **Ctrl+backtick and Ctrl+Space:** most terminals cannot tell Ctrl+backtick
> apart from Ctrl+Space — both send the NUL byte, which tmux calls `C-Space`.
> The config binds `C-Space`, so either key acts as the prefix. Plain backticks
> type normally in the shell, nvim, and the agent.

> **Why not `Super+g`?** Terminals do not send the Super/Windows key to tmux in
> a usable form, so tmux cannot bind it portably. Use the prefix bindings, or
> remap Super in your terminal emulator to send an escape sequence.

## Config

`~/.config/idemux/config`, sourced as shell code:

```sh
# Directories searched for git repositories.
IDE_ROOTS=(
  /home/you/code
  /home/you/work
)

# Command run in the left pane.
IDE_EDITOR=nvim

# Command run in the right pane.
IDE_AGENT='claude'

# Toggled into the left pane with prefix+g.
IDE_LAZYGIT=lazygit

# How many directory levels below each root to search.
IDE_SEARCH_DEPTH=3
```

Commands are run through `$SHELL -lc`, so arguments and shell syntax work:

```sh
IDE_AGENT='aider --model sonnet'
IDE_EDITOR='nvim -O'
```

- `idemux setup` re-runs the wizard (it offers to keep the current roots).
- `idemux config` opens the file in `$VISUAL`/`$EDITOR`.
- Config changes are picked up the next time an `idemux` subcommand runs.
- `idemux.conf` changes are applied by `prefix+r` (or the next `idemux` launch).

## Commands

| Command                  | Description                                              |
|--------------------------|----------------------------------------------------------|
| `idemux`                 | Start or attach (wizard + picker on first launch)        |
| `idemux open <repo>...`  | Open repos, starting the session if needed (no attach)   |
| `idemux pick`            | fzf picker against a running session (`prefix+o`)        |
| `idemux repos`           | Print the repos discovered under the configured roots    |
| `idemux close [window]`  | Close a repo window (`prefix+X`)                         |
| `idemux toggle-left`     | Swap editor/lazygit (`prefix+g`)                         |
| `idemux toggle-right`    | Swap agent/shell (`prefix+a`)                            |
| `idemux reload`          | Re-source the tmux config and clean up (`prefix+r`)      |
| `idemux setup`           | Re-run the config wizard                                 |
| `idemux config`          | Edit the config file                                     |
| `idemux help`            | Show usage                                               |

## How it works

- **Dedicated server.** `idemux` uses `tmux -L idemux`, so none of these keys or
  options leak into your normal tmux sessions. Attach manually with
  `tmux -L idemux attach -t idemux`; reset everything with
  `tmux -L idemux kill-server`.
- **Repo window.** Each repo gets a window named after its directory (duplicate
  names are prefixed with the parent directory). The window option `@idemux_repo`
  records the absolute path, so opening the same repo again focuses the
  existing window instead of creating a duplicate.
- **State-preserving toggles.** Hidden panes live in a detached `_idemux_parking`
  session. Toggling joins the alternate pane into the repo window and breaks
  the visible one out. Both processes keep running; nvim keeps its buffers and
  the agent keeps its conversation.
- **Close.** Closing a repo kills its window and any parked panes belonging to
  it. Orphaned parked panes (for example, after killing a window with tmux
  itself) are cleaned up on the next launch or reload.
- **Discovery.** Repos are found by locating `.git` entries (directories or
  worktree files) up to `IDE_SEARCH_DEPTH` levels below each root. Common build
  and dependency directories (`node_modules`, `.venv`, `target`, `dist`,
  `build`, `vendor`, ...) are skipped. The scan runs live every time the picker
  opens.

## Notes

- If the editor, lazygit, or agent exits, that pane drops to a login shell so
  the window keeps its layout. Run the tool again in that pane, or toggle and
  use the other side.
- Running `idemux` from inside another tmux server attaches a nested client.
- The status bar and window list show only repo windows; the parking session is
  separate.
- Overridable environment variables (mostly useful for testing):
  `IDEMUX_CONFIG`, `IDEMUX_CONF`, `IDEMUX_SERVER`, `IDEMUX_SESSION`,
  `IDEMUX_PARKING`.

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `no config at ...` | Run `idemux setup`. |
| `tmux >= 3.2 is required` | Upgrade tmux. |
| Picker is empty | Run `idemux repos`; check the roots and `IDE_SEARCH_DEPTH`. |
| Key does nothing | Make sure you are on the `idemux` server, then run `idemux reload`. |
| Stale session or panes | `tmux -L idemux kill-server`, then `idemux` again. |
| Want a fresh config | Delete `~/.config/idemux/config` and run `idemux`. |

## License

Released under the [MIT License](LICENSE).

Copyright (c) 2026 Barnaby O'Callaghan.
