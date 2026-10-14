# tmux and Other Terminal Multiplexers: A Practical Guide

A **terminal multiplexer** lets you run multiple terminal sessions inside a single window, and, more importantly, lets those sessions **keep running when you disconnect** and be reattached later. They are essential for remote server work, long-running jobs, and organised terminal workflows.

---

## 1. Why Use a Multiplexer?

| Problem | How a multiplexer helps |
|---|---|
| SSH connection drops and kills your running job | Session keeps running on the server; reattach after reconnecting |
| Need several shells on one SSH connection | Windows and panes inside one connection |
| Want to watch logs while editing code | Split panes side by side |
| Pair programming or teaching remotely | Two users attach to the same session |
| Want a repeatable workspace layout | Scripted or saved sessions |
| Closing the terminal window by accident | Work survives; just reattach |

---

## 2. The Tools at a Glance

| Tool | Category | Main use | Config |
|---|---|---|---|
| **tmux** | Multiplexer | The standard: sessions, windows, panes, scripting, plugins | `~/.tmux.conf` |
| **GNU Screen** | Multiplexer | The original; ubiquitous on old or minimal systems; serial console support | `~/.screenrc` |
| **Zellij** | Multiplexer | Modern, beginner-friendly, on-screen key hints, layouts, plugin system (WebAssembly) | KDL files in `~/.config/zellij/` |
| **Byobu** | Wrapper | Friendly front end over tmux (or screen) with status bar and F-key shortcuts | `~/.byobu/` |
| **dtach / abduco** | Session detacher | Minimal: detach/reattach a single program only, no windows or panes | none |
| **tmuxinator / tmuxp / smug** | Session managers | Define project layouts in YAML and launch them with one command | YAML files |
| **mosh** | Roaming remote shell | SSH replacement that survives IP changes and high latency (complements a multiplexer) | none |
| **reptyr** | Utility | "Steal" a running process from its terminal into your tmux/screen | none |
| **nohup / disown / `&`** | Shell basics | Keep a single command alive after logout (no reattach) | none |
| **systemd-run / at / cron** | Job runners | Run scheduled or unattended tasks properly, not interactively | various |
| **Terminal emulators with built-in splits** (iTerm2, Windows Terminal, Kitty, WezTerm, Terminator, Tilix) | Local GUI | Tabs and splits locally; no persistence over SSH (WezTerm and Kitty can do some multiplexing) | app-specific |

**Rule of thumb:** local convenience → a good terminal emulator; remote, persistent, scriptable → tmux (or Zellij if you prefer discoverability).

---

## 3. tmux

### 3.1 Core Concepts

```
Server
 └── Session        (a named workspace; can be detached)
      └── Window    (like a tab)
           └── Pane (a split region with its own shell)
```

- The **server** runs in the background and holds everything.
- Your terminal is a **client** attached to a session.
- The default **prefix** key is `Ctrl+b`. Most shortcuts are: press the prefix, release, then press the key.

### 3.2 Install

```bash
sudo apt install tmux        # Debian/Ubuntu
sudo dnf install tmux        # Fedora/RHEL
brew install tmux            # macOS
pacman -S tmux               # Arch
```

### 3.3 Session Commands (from the shell)

```bash
tmux                           # new unnamed session
tmux new -s work               # new named session
tmux new -s work -d            # create detached
tmux ls                        # list sessions
tmux attach -t work            # attach (alias: a)
tmux a                         # attach to the most recent
tmux kill-session -t work
tmux kill-server               # kill everything
tmux new -A -s main            # attach if exists, otherwise create (handy)
```

### 3.4 Essential Key Bindings (prefix = `Ctrl+b`)

**Sessions**

| Keys | Action |
|---|---|
| `d` | Detach from session |
| `s` | Interactive session list/switcher |
| `$` | Rename session |
| `(` / `)` | Previous / next session |

**Windows**

| Keys | Action |
|---|---|
| `c` | Create window |
| `,` | Rename window |
| `n` / `p` | Next / previous window |
| `0`–`9` | Jump to window by number |
| `w` | Choose window from a list |
| `&` | Close window (confirm) |
| `l` | Last used window |

**Panes**

| Keys | Action |
|---|---|
| `%` | Split left/right |
| `"` | Split top/bottom |
| Arrow keys | Move between panes |
| `o` | Cycle to next pane |
| `z` | Zoom/unzoom current pane |
| `x` | Close pane |
| `{` / `}` | Swap pane with previous / next |
| `Space` | Cycle layouts |
| `Ctrl+Arrow` | Resize pane (hold prefix or repeat) |
| `q` | Show pane numbers |
| `!` | Break pane out into its own window |

**Copy mode and misc**

| Keys | Action |
|---|---|
| `[` | Enter copy/scroll mode (scroll with arrows/PageUp) |
| `q` | Exit copy mode |
| `]` | Paste buffer |
| `:` | Command prompt |
| `?` | List all key bindings |
| `t` | Show clock |

### 3.5 Copy Mode

```
prefix + [       enter copy mode
Space            start selection   (vi mode)
Enter            copy and exit
prefix + ]       paste
/  or  ?         search forward / backward
```

Enable vi-style keys and mouse support in config (below). For system clipboard integration, use `tmux-yank` or `set -g set-clipboard on` with a supporting terminal.

### 3.6 A Solid Starter `~/.tmux.conf`

```tmux
# Better prefix (Ctrl+a), keep Ctrl+b free for other tools
unbind C-b
set -g prefix C-a
bind C-a send-prefix

# Sensible defaults
set -g mouse on
set -g history-limit 50000
set -g base-index 1                 # windows start at 1
setw -g pane-base-index 1
set -g renumber-windows on
set -sg escape-time 10              # faster Esc in vim
set -g default-terminal "tmux-256color"
set -as terminal-features ",*:RGB"  # true color
setw -g mode-keys vi

# Intuitive splits (keep current directory)
bind | split-window -h -c "#{pane_current_path}"
bind - split-window -v -c "#{pane_current_path}"
bind c new-window -c "#{pane_current_path}"
unbind '"'
unbind %

# Vim-style pane navigation
bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R

# Reload config
bind r source-file ~/.tmux.conf \; display "Config reloaded"

# Status bar
set -g status-position bottom
set -g status-interval 5
set -g status-left  "#[bold]#S "
set -g status-right "%Y-%m-%d %H:%M"
```

Apply without restarting: `tmux source-file ~/.tmux.conf` (or the `r` binding above).

### 3.7 Plugins (TPM: Tmux Plugin Manager)

```bash
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
```
Add to `~/.tmux.conf`:
```tmux
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-sensible'
set -g @plugin 'tmux-plugins/tmux-resurrect'    # save/restore sessions across reboots
set -g @plugin 'tmux-plugins/tmux-continuum'    # automatic saving
set -g @plugin 'tmux-plugins/tmux-yank'         # clipboard integration
run '~/.tmux/plugins/tpm/tpm'
```
Install plugins with `prefix + I`; update with `prefix + U`.

### 3.8 Scripting tmux

```bash
#!/usr/bin/env bash
SESSION=dev
tmux has-session -t $SESSION 2>/dev/null && exec tmux attach -t $SESSION

tmux new-session  -d -s $SESSION -n editor -c ~/project
tmux send-keys    -t $SESSION:editor 'nvim .' C-m
tmux new-window   -t $SESSION -n server -c ~/project
tmux send-keys    -t $SESSION:server 'npm run dev' C-m
tmux split-window -h -t $SESSION:server -c ~/project
tmux send-keys    -t $SESSION:server.2 'tail -f logs/app.log' C-m
tmux select-window -t $SESSION:editor
tmux attach -t $SESSION
```

Useful scripting commands: `send-keys`, `split-window`, `new-window`, `select-pane`, `capture-pane -p` (read pane output), `run-shell`, `display-message`.

### 3.9 Sharing a Session (Pair Programming)

Same user account (for example, both SSH into one shared login):
```bash
tmux new -s pair
# other person:
tmux attach -t pair
```
For separate accounts, use a shared socket:
```bash
tmux -S /tmp/pair new -s pair
chmod 777 /tmp/pair          # or use a shared group
# other user:
tmux -S /tmp/pair attach -t pair
```
Use `attach -r` for read-only viewing. Only share sessions with people you trust, since they get shell access as you.

### 3.10 Nested tmux (tmux inside tmux over SSH)

Commonly you run tmux locally and on a remote host. Inner and outer both respond to the same prefix. Options:
- Use a different prefix on the local one (for example `Ctrl+a` local, `Ctrl+b` remote).
- Press the prefix **twice** to send it to the inner session (`prefix` + `prefix` when `bind C-a send-prefix` is set).
- Only run tmux on the remote, and use a plain terminal locally.

---

## 4. GNU Screen

The veteran multiplexer (since 1987); installed by default on many older or minimal systems.

```bash
screen                        # new session
screen -S work                # named
screen -ls                    # list
screen -r work                # reattach
screen -d -r work             # detach elsewhere then reattach
screen -dmS job command       # run a command detached
screen /dev/ttyUSB0 115200    # serial console, a classic use case
```

Prefix is `Ctrl+a`.

| Keys | Action |
|---|---|
| `Ctrl+a c` | New window |
| `Ctrl+a n` / `p` | Next / previous window |
| `Ctrl+a "` | Window list |
| `Ctrl+a d` | Detach |
| `Ctrl+a S` | Horizontal split |
| `Ctrl+a \|` | Vertical split |
| `Ctrl+a Tab` | Move focus between regions |
| `Ctrl+a X` | Close region |
| `Ctrl+a [` | Copy/scroll mode |
| `Ctrl+a k` | Kill window |
| `Ctrl+a \` | Quit screen entirely |

**tmux vs Screen:** tmux has a cleaner scripting interface, better pane handling, active development, and a bigger plugin community. Screen wins when it is the only thing installed, and for serial-line connections.

---

## 5. Zellij

A modern multiplexer written in Rust, designed to be usable without memorising key bindings.

```bash
cargo install --locked zellij      # or: brew install zellij, or a distro package
zellij                             # start
zellij attach -c work              # attach or create named session
zellij ls
zellij --layout dev.kdl
```

Highlights:
- **Status bar shows available keys** in the current mode; very gentle learning curve.
- **Modes** (Pane, Tab, Resize, Scroll, Session, Locked) rather than a prefix key; `Ctrl+g` toggles locked mode so keys pass through to the app.
- **Floating panes** and stacked panes.
- **Layouts** in KDL config files.
- **WebAssembly plugin system**.
- Built-in session resurrection and a web client in recent versions.

| Keys (default) | Action |
|---|---|
| `Ctrl+p` then `n` | New pane |
| `Ctrl+p` then `d` / `r` | New pane down / right |
| `Ctrl+t` then `n` | New tab |
| `Ctrl+o` then `d` | Detach |
| `Ctrl+s` | Scroll mode |
| `Alt+arrows` / `Alt+hjkl` | Move between panes |
| `Ctrl+q` | Quit |

Example layout (`dev.kdl`):
```kdl
layout {
    pane split_direction="vertical" {
        pane command="nvim"
        pane split_direction="horizontal" {
            pane command="npm" { args "run" "dev"; }
            pane
        }
    }
}
```

Trade-off: newer and heavier than tmux, less ubiquitous on servers, but great defaults.

---

## 6. Byobu

A wrapper that makes tmux (default) or Screen approachable on Ubuntu/Debian.

```bash
sudo apt install byobu
byobu-enable        # auto-start on login
byobu
```

- Function-key shortcuts: `F2` new window, `F3`/`F4` previous/next window, `F6` detach, `F7` scroll mode, `F8` rename, `F9` config menu.
- Shows system status (CPU, memory, updates, uptime) in the status bar.
- Good for people who want persistence without learning tmux keybindings.

---

## 7. Lightweight Persistence: dtach, abduco, nohup

When you only need to keep **one program** alive and reconnect to it, not a whole workspace.

### dtach / abduco
```bash
dtach -c /tmp/mysession bash        # create and attach
# detach with Ctrl+\ (dtach default)
dtach -a /tmp/mysession             # reattach

abduco -c job long-command          # create
abduco -a job                       # attach
abduco                              # list
```
No windows, panes, or scrollback handling; it only passes data through, so use your terminal's own scrolling. They are often combined with `dvtm` (a tiling window manager for the console).

### nohup, disown, and `&`
```bash
nohup ./script.sh > out.log 2>&1 &      # survives logout, output to file
./script.sh &  disown                   # detach a job from the shell
setsid ./script.sh &                    # new session, fully detached
```
You **cannot reattach** to these later to interact; you only get logs. Fine for fire-and-forget tasks.

### reptyr
Forgot to start tmux before a long job? Start tmux, then:
```bash
reptyr <pid>
```
to move the running process into your current terminal. (May require `sudo sysctl kernel.yama.ptrace_scope=0` on some systems.)

---

## 8. Session Managers: tmuxinator, tmuxp, smug

Automate layouts for each project.

**tmuxinator** (Ruby), `~/.config/tmuxinator/myproject.yml`:
```yaml
name: myproject
root: ~/code/myproject
windows:
  - editor: nvim .
  - server:
      layout: main-vertical
      panes:
        - npm run dev
        - tail -f log/dev.log
  - shell:
```
```bash
tmuxinator start myproject
```

**tmuxp** (Python): `tmuxp load project.yaml`, with JSON/YAML config and freeze support (`tmuxp freeze` exports the current session).

**smug** (Go): single binary, YAML config, `smug start myproject`.

---

## 9. mosh: Not a Multiplexer, but a Great Partner

**mosh** (mobile shell) replaces the SSH *transport* for interactive shells:
- Survives Wi-Fi changes, laptop sleep, and IP changes.
- Local echo for snappy typing on slow links.
- Uses UDP (ports 60000–61000 by default) and starts through SSH.

```bash
mosh user@server
mosh user@server -- tmux new -A -s main
```

mosh keeps the *connection* alive, but it has limited scrollback and no splits. Combine it with tmux for the best experience.

---

## 10. Comparison

| Feature | tmux | Screen | Zellij | Byobu | dtach/abduco |
|---|---|---|---|---|---|
| Detach/reattach | Yes | Yes | Yes | Yes | Yes |
| Windows/tabs | Yes | Yes | Yes | Yes | No |
| Panes/splits | Yes | Yes (basic) | Yes (+floating) | Yes | No |
| Scripting | Excellent | Limited | Layouts, plugins | via tmux | n/a |
| Plugin ecosystem | Large (TPM) | Small | Growing (WASM) | Small | None |
| Learning curve | Medium | Medium | Low | Low | Very low |
| Default availability on servers | Common | Very common | Rare | Ubuntu repos | Rare |
| Mouse support | Yes (opt-in) | Limited | Yes | Yes | No |
| Resource use | Low | Very low | Moderate | Low | Minimal |
| Active development | Yes | Slow | Yes | Yes | Minimal |

---

## 11. Choosing the Right Tool

| If you... | Use |
|---|---|
| Work on remote servers daily and want power and scriptability | **tmux** |
| Are on a minimal or legacy box, or need serial console | **Screen** |
| Want a modern, discoverable experience with sensible defaults | **Zellij** |
| Want persistence with minimal learning on Ubuntu | **Byobu** |
| Just need one long job to survive disconnects | **dtach / abduco / nohup** |
| Forgot to use tmux and the job is already running | **reptyr** |
| Have an unreliable or roaming network connection | **mosh + tmux** |
| Switch between many projects with fixed layouts | **tmuxinator / tmuxp / smug** |
| Only need tabs and splits locally | **A terminal emulator** (iTerm2, Windows Terminal, WezTerm, Kitty) |
| Need scheduled or unattended jobs | **cron, systemd timers, `at`**, not a multiplexer |

---

## 12. Quick Start Workflow (tmux)

```bash
ssh user@server
tmux new -A -s main          # create or attach
# Ctrl+b %  -> split, run your editor on one side and a build/logs on the other
# Ctrl+b c  -> new window for another task
# Ctrl+b d  -> detach when leaving
# ...connection drops, or you go home...
ssh user@server
tmux a -t main               # everything is exactly as you left it
```

---

## 13. Tips and Troubleshooting

| Issue | Fix |
|---|---|
| Colors look wrong | Set `default-terminal "tmux-256color"` and enable RGB features; make sure your terminal's `TERM` is correct |
| Esc key is slow in vim | `set -sg escape-time 10` |
| Can't scroll with mouse wheel | `set -g mouse on` |
| Copy/paste to system clipboard not working | Use `tmux-yank`, `set -g set-clipboard on`, and a terminal that supports OSC 52; hold `Shift` to select with the terminal's native selection when mouse mode is on |
| "sessions should be nested with care" | You're inside tmux already; detach first or unset `TMUX` (`TMUX= tmux`) |
| Config changes not applying | Reload with `tmux source-file ~/.tmux.conf`; some options need a new server |
| Panes lose their working directory | Use `-c "#{pane_current_path}"` in split/new-window bindings |
| Lost sessions after reboot | Use `tmux-resurrect` + `tmux-continuum` |
| Key conflict with another app (Ctrl+b in vim, Ctrl+a in readline) | Change the prefix or send it through with `prefix prefix` |
| Output of a long command scrolled away | Increase `history-limit`; use `prefix + [` then search |

Handy commands:
```bash
tmux list-keys                     # all bindings
tmux show-options -g               # global options
tmux capture-pane -pS -3000 > out.txt   # dump scrollback to a file
tmux display-message -p '#S'       # print current session name
```

---

## 14. Key Takeaways

1. **Terminal multiplexers provide persistence**: start work, detach, and resume from anywhere.
2. **tmux** is the general-purpose standard; learn prefix, sessions, windows, panes, and a small config.
3. **Screen** remains useful on minimal systems and serial consoles.
4. **Zellij** and **Byobu** lower the learning curve.
5. **dtach, abduco, nohup, and reptyr** cover the simple cases: keeping a single process alive.
6. **mosh** fixes flaky networks; **tmuxinator/tmuxp/smug** automate project layouts.
7. For purely local tabs and splits, your terminal emulator may be all you need.

---

*Further reading: `man tmux`, the [tmux wiki](https://github.com/tmux/tmux/wiki), the [Zellij docs](https://zellij.dev/documentation/), and `man screen`.*
