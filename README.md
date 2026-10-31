# tmux-ide

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

The left pane takes 60% of the window width; the right pane takes the remaining
40%. Every pane starts in the repository root. The hidden side of each toggle
keeps running in a parking session, so switching between nvim and lazygit (or
between the agent and a shell) does not lose state.

## Requirements

- Linux (the repo scan uses GNU `find -printf`)
- tmux >= 3.2
- `git` and `fzf`
- an editor, `lazygit`, and a coding-agent CLI
- bash 4.4+ (arrays, `mapfile -d`)

## Install

Keep the script in the repository; it finds `tmux-ide.conf` relative to itself.

```sh
git clone <this-repo> ~/gh/tmux-ide
ln -s ~/gh/tmux-ide/bin/ide ~/.local/bin/ide
```

Make sure `~/.local/bin` is on your `PATH`.

## First run

```sh
ide
```

1. If `~/.config/tmux-ide/config` does not exist, a small wizard asks for:
   - one or more directories to search for git repositories
   - the editor command (default `nvim`)
   - the coding-agent command (e.g. `claude`, `aider`, `codex`)
2. It scans those roots and opens an fzf picker. `TAB` multi-selects, `ENTER`
   opens one window per selected repository.
3. It creates a **dedicated** tmux server (`tmux -L tmux-ide`) and attaches.

On later runs `ide` attaches to the existing session without showing the picker.
If the IDE session is gone (for example, you closed the last repo), the picker
appears again.

## Keys

The dedicated server uses a bare backtick (`` ` ``) as its prefix. The blue
status bar sits at the top and shows these bindings on the right as a cheat
sheet.

| Key        | Action                                      |
|------------|---------------------------------------------|
| `prefix+g` | left pane: editor <-> lazygit               |
| `prefix+a` | right pane: agent <-> shell                 |
| `prefix+o` | open more repos with fzf                    |
| `prefix+X` | close the current repo (with confirmation)  |
| `prefix+R` | reload `tmux-ide.conf` and clean up panes   |

`prefix+x` still kills a pane, and the normal tmux keys are untouched.

> **Typing backticks:** because the prefix is a printable key, tmux intercepts
> every backtick — including inside nvim, the shell, and the agent. Press it
> twice to send one literal backtick to the pane. This is the trade-off for a
> single-key prefix; switch back to `C-a` or `C-b` in `tmux-ide.conf` if it gets
> in the way.

> **Why not `Super+g`?** Terminals do not send the Super/Windows key to tmux in
> a usable form, so tmux cannot bind it portably. Use the prefix bindings, or
> remap Super in your terminal emulator to send an escape sequence.

## Config

`~/.config/tmux-ide/config`, sourced as shell code:

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

- `ide setup` re-runs the wizard (it offers to keep the current roots).
- `ide config` opens the file in `$VISUAL`/`$EDITOR`.
- Config changes are picked up the next time an `ide` subcommand runs.
- `tmux-ide.conf` changes are applied by `prefix+R` (or the next `ide` launch).

## Commands

| Command                 | Description                                              |
|-------------------------|----------------------------------------------------------|
| `ide`                   | Start or attach (wizard + picker on first launch)        |
| `ide open <repo>...`    | Open repos, starting the session if needed (no attach)   |
| `ide pick`              | fzf picker against a running session (`prefix+o`)        |
| `ide repos`             | Print the repos discovered under the configured roots    |
| `ide close [window]`    | Close a repo window (`prefix+X`)                         |
| `ide toggle-left`       | Swap editor/lazygit (`prefix+g`)                         |
| `ide toggle-right`      | Swap agent/shell (`prefix+a`)                            |
| `ide reload`            | Re-source the tmux config and clean up (`prefix+R`)      |
| `ide setup`             | Re-run the config wizard                                 |
| `ide config`            | Edit the config file                                     |
| `ide help`              | Show usage                                               |

## How it works

- **Dedicated server.** `ide` uses `tmux -L tmux-ide`, so none of these keys or
  options leak into your normal tmux sessions. Attach manually with
  `tmux -L tmux-ide attach -t ide`; reset everything with
  `tmux -L tmux-ide kill-server`.
- **Repo window.** Each repo gets a window named after its directory (duplicate
  names are prefixed with the parent directory). The window option `@ide_repo`
  records the absolute path, so opening the same repo again focuses the
  existing window instead of creating a duplicate.
- **State-preserving toggles.** Hidden panes live in a detached `_ide_parking`
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
- Running `ide` from inside another tmux server attaches a nested client.
- The status bar and window list show only repo windows; the parking session is
  separate.
- Overridable environment variables (mostly useful for testing):
  `TMUX_IDE_CONFIG`, `TMUX_IDE_CONF`, `TMUX_IDE_SERVER`, `TMUX_IDE_SESSION`,
  `TMUX_IDE_PARKING`.

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `no config at ...` | Run `ide setup`. |
| `tmux >= 3.2 is required` | Upgrade tmux. |
| Picker is empty | Run `ide repos`; check the roots and `IDE_SEARCH_DEPTH`. |
| Key does nothing | Make sure you are on the `tmux-ide` server, then run `ide reload`. |
| Stale session or panes | `tmux -L tmux-ide kill-server`, then `ide` again. |
| Want a fresh config | Delete `~/.config/tmux-ide/config` and run `ide`. |
