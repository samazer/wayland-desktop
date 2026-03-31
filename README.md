<!-- SPDX-FileCopyrightText: 2026 Sam Azer <sam dot azer at azertech dot net>  -->
<!-- SPDX-License-Identifier: GPL-3.0-or-later  -->
# wayland-desktop — KDE Plasma 6 Desktop Setup toolset

A set of command-line tools for programmatic Desktop Setup on boot
into KDE Plasma6 on Wayland.

These tools are a little more capable than the Session Restore
functionality that was available on X11.

A list of programs can be launched, each window moved to any screen on
any desktop and fed keystrokes as needed to restore, after a reboot,
even a complicated workstation setup.

There is also a Desktop Snapshot tool: Just setup your desktop the way
you want it and run the snapshot tool to generate a script to restore
(most of) that desktop.  See the [Limitations](#limitations), below, for
information about things that don't work as well as desired.

Note that some details involved in Desktop Setup are not as easy as 
desired.  Mouse clicks are not working properly yet and
there is no support, yet, for Plasma6 activities.)

---

## Available Documents
| Document                                     | Description                                                                          |
|:---------------------------------------------|:-------------------------------------------------------------------------------------|
| [docs/OVERVIEW.md](docs/OVERVIEW.md)         | How the library works                                                                |
| [docs/INSTALL.md](docs/INSTALL.md)           | Insetallation and setup information.                                                 |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | System context and design overview                                                   |
| [docs/PROBLEMS.md](docs/PROBLEMS.md)         | Known limitations and open issues                                                    |
| [WARNINGS.md](WARNINGS.md) | User-level Usage Warnings |
| [docs/GLOSSARY.md](docs/GLOSSARY.md)         | Definitions of terms used in the codebase and documentation                          |
| [docs/ADRs.md](docs/ADRs.md)                 | Architectural Decision Records                                                       |
| [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md) | Project requirements and implementation status                                       | [docs/DATA-MODELS.md](docs/DATA-MODELS.md) | WindowInfo, DesktopInfo, ScreenInfo dataclasses |
| [docs/LIBRARY-API.md](docs/LIBRARY-API.md)   | Python API reference (KWinSession, WindowProbe, WindowManager, DesktopManager)       |
| [docs/CONFIG.md](docs/CONFIG.md)             | Configuration file for all wayland-desktop scripts                                   |
| [docs/SNAPSHOT.md](docs/SNAPSHOT.md)         | Documentation for the Desktop Setup (YAML) file                                      |
| [ABOUT.md](ABOUT.md)         | Some background information about this project |


---

## Contents

0. [Copyright &amp; License](#copyright--license)
1. [Command-line tools](#command-line-tools)
   - [restore](#restore)
   - [snapshot](#snapshot)
   - [realign](#realign)
   - [mouse-clicks](#mouse-clicks)
   - [probe](#probe)
   - [getconfig](#getconfig)
   - [secret-service](#secret-service)
2. [Limitations](#limitations)
3. [Troubleshooting](#troubleshooting)

---

## Copyright &amp; License

<!-- do not edit: begin -->

**Wayland Desktop Setup**
**Plasma6 Session Restore for Wayland, including input injection.**

Copyright © 2026 by Sam Azer&lt;sam dot azer at azertech dot net&gt;, All rights reserved.

This software is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or (at
your option) any later version.

This software is distributed in the hope that it will be useful, but
WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the [GNU
General Public License](LICENSE.md) for more details.

You should have received a copy of the [GNU General Public
License](LICENSE.md) along with this program.  If not, see
<https://www.gnu.org/licenses/>.

**Wayland Desktop Setup (documentation)**

Wayland Desktop Restore (documentation) Copyright © 2026 by Sam Azer &lt;sam dot azer at azertech dot net&gt; is licensed under Creative Commons Attribution-ShareAlike 4.0 International. To view a copy of this license, visit https://creativecommons.org/licenses/by-sa/4.0/

#### Creative Commons Attribution-ShareAlike 4.0 International
This license requires that re-users give credit to the creator. It allows re-users to distribute, remix, adapt, and build upon the material in any medium or format, even for commercial purposes. If others remix, adapt, or build upon the material, they must license the modified material under identical terms.

- Credit must be given to the creator.
- Adaptations must be shared under the same terms.

[See the full License](https://creativecommons.org/licenses/by-sa/4.0/)

<!-- do not edit: end -->

## Command-line tools

### restore

The recommended entry point for a full desktop restore.  Presents two startup
confirmation dialogs via `kdialog`, runs `restore-user-before` if present,
hands off to `kwin_restore.py`, then runs `restore-user-after` if present.

Set up as a shell alias pointing at the script in your wayland-desktop directory:

```bash
alias restore="/path/to/wayland-desktop/restore"
```

**Usage:**

```
restore [OPTIONS] [APP]
```

**Arguments:**

| Argument | Description |
|---|---|
| `APP` | *(optional)* Restore only this one application (alias or `resourceClass`). Equivalent to `--filter APP`. |

**Options:**

| Option | Default | Description |
|---|---|---|
| `--setup FILE` | see below | Snapshot file to restore from |
| `--config FILE` | `~/.config/wayland-desktop/config.yaml` | Alternate config |
| `--log FILE` | | Tee all output to FILE (ANSI codes stripped). Parent directories are created as needed. |
| `--dry-run` | | Show what would be done, without doing it. Also bypasses the startup dialogs. |
| `--no-launch` | | Only place already-open windows; don't launch new ones |
| `--timeout SECS` | 30 | Per-app launch timeout |
| `--delay MS` | 500 | Milliseconds between app launches in phase 1 |
| `--settle-delay SECS` | 3.0 | Seconds between phase 2 (move) and phase 3 (finalise) |
| `--phase5-delay MS` | 400 | Milliseconds to wait after each desktop switch in phase 5 |
| `--filter APP` | | Only restore entries matching this alias or `resourceClass` (repeatable) |
| `--verbose` | | Print a debug line after every KWin script call |
| `--show-window` | | Show the progress window regardless of config setting |
| `--no-window` | | Suppress the progress window regardless of config setting |
| `--show-clicks` | | ⚠️ Not currently working. Intended to show a dot at each injected pointer position during phase 4. |

**Dialog behaviour:**

1. **Ready to restore?** — Confirms you are ready to proceed and reminds you
   that you will need to unlock your password manager if you use any {PASS}
   tokens in your desktop setup input_actions.  Note that, so far, the 
   Secret Service protocol has only been tested with KeepassXC.

   This dialog is skipped when `--dry-run` is in the command line.

2. Optionally run a user-script to perform any needed actions before the desktop
   setup.

   **`restore-user-before`:** Checks for an executable script called 
   `restore-user-before` in the `user_scripts_folder` directory
   (configured in `config.yaml`, default `~/.config/wayland-desktop/scripts`).
   If found, it is run before the main Desktop Setup begins.  The `getconfig` 
   and `secret-service` tools are available inside `restore-user-before` (though
   the Secret Service password tool must be unlocked before use.)

3. **Watch out for pop-ups** — Reminds you that portal authorization dialogs
   (for input injection) and Secret Servie approval dialogs (for secure password 
   access) may pop-up.  Also warns that such windows may appear beneath other 
   windows.

   Cancel exits cleanly without running the restore.
   
   Both dialogs are skipped when `--dry-run` is in the command line.

4. **`restore-user-after`:** After the desktop setup is completed, the `restore` 
   script checks for an executable script called `restore-user-after` in the same 
   `user_scripts_folder` directory.  If found, it is run regardless of whether 
   the desktop setup succeeded or failed.  The exit code of `kwin_restore.py` is 
   preserved and re-applied after the hook returns.  Use it for post-restore 
   tasks such as any clean-up that may be required.

**Setup file resolution:**

1. If `--setup FILE` is given, that path is used directly (`~` is expanded).
2. Otherwise, `default_setup` from `config.yaml` is read and searched in order:
   - The current working directory
   - `~/.config/wayland-desktop/`
   - `snapshot_dir` (as configured in `config.yaml`)
3. If the file cannot be found, the script exits with a clear error message.

Set `default_setup` in `config.yaml` once to avoid specifying it on every run:

```yaml
default_setup: "setup.yaml"
```

**Examples:**

```bash
# Full restore using the default setup file
restore

# Restore from an explicit snapshot file
restore --setup ~/.config/wayland-desktop/snapshots/2026-03-17_2130.yaml

# Dry run (no dialogs, nothing changed)
restore --dry-run

# Restore only Thunderbird
restore --filter thunderbird

# Restore a single app by positional shorthand
restore chrome

# Log the full run to a file
restore --log ~/.config/wayland-desktop/restore.log
```

**Single-instance behaviour:**

When `single_instance: true` in a snapshot entry, the restore script checks
whether a window with that `resourceClass` is already open.  If it is, it
places the existing window without launching a new copy.  If it is not open, it
launches it.

When `single_instance: false` (the default), a new instance is always launched
regardless of whether one is already open.  This is the correct behaviour for
Konsole and Dolphin, which you may want running on multiple desktops
simultaneously.

**Automating on login:**

The easiest way to configure autostart is through **System Settings → Autostart**
(search for "Autostart" in KDE System Settings).  From there you can add any
script or application to run at login.

Recommended autostart setup — add two entries:

1. **Your Secret Service manager** (e.g. KeePassXC) — add it to Autostart so
   it is running and unlocked before the restore begins.  If you use `{PASS}`
   tokens in `input_actions`, KeePassXC must be open with its database unlocked
   before the restore reaches phase 4.

2. **The restore script** — add `restore` (or the full path to it) as an
   autostart application.

Alternatively, create `~/.config/autostart/wayland-desktop-restore.desktop`
manually:

```ini
[Desktop Entry]
Type=Application
Name=Wayland Desktop Restore
Exec=/path/to/wayland-desktop/restore
X-KDE-AutostartPhase=2
X-KDE-StartupNotify=false
```

`AutostartPhase=2` ensures KWin is fully initialised before the script runs.

The `restore` wrapper will:
- Show the two startup confirmation dialogs
- Run `restore-user-before` (from `user_scripts_folder`) if found — use this
  to start any services the restore depends on that are not already covered
  by the Autostart entries above
- Launch `kwin_restore.py` to perform the actual window placement
- Run `restore-user-after` (from `user_scripts_folder`) if found — use this
  for any post-restore tasks

To add a log file to your autostart entry:

```ini
Exec=/path/to/wayland-desktop/restore --log ~/.config/wayland-desktop/restore.log
```

For a fully non-interactive autostart (no dialogs), invoke `restore`
directly:

```ini
Exec=/usr/bin/python3 /path/to/wayland-desktop/wayland/restore
```

---

### snapshot

Capture the current window layout to a YAML file.

```bash
snapshot [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--output FILE` | auto-named in `snapshot_dir` | Where to save the snapshot |
| `--include-system` | | Include compositor windows (normally excluded) |
| `--config FILE` | `~/.config/wayland-desktop/config.yaml` | Alternate config |
| `--show-config` | | Print loaded aliases and exit |
| `--dry-run` | | Print YAML to stdout without saving to disk |

**Examples:**

```bash
# Check what aliases are loaded
snapshot --show-config

# Preview the snapshot YAML without saving
snapshot --dry-run

# Save to auto-named file (e.g. ~/.config/wayland-desktop/snapshots/2026-03-17_2130.yaml)
snapshot

# Save to a specific file
snapshot --output ~/my-layout.yaml
```

The script always prints the complete snapshot YAML to stdout immediately after
capture, whether or not it saves to disk.  This makes it easy to review the
output before committing to a file, and to diff two captures side by side.

Within each desktop section of the snapshot, windows are sorted alphabetically
by application alias.  This ordering is stable across captures, so `diff` on
two snapshots produces only meaningful differences rather than noise from
arbitrary capture ordering.

**Snapshot file format:**

```yaml
meta:
  timestamp: '2026-03-17T21:30:00'
  hostname: m68.int.azer.ca

windows:
  # ============
  # | Plumbing |
  # ============

  - app: thunderbird
    resource_class: thunderbird-esr
    desktop_filename: thunderbird-esr
    desktop: Plumbing          # alias (or canonical name if no alias defined)
    screen: main               # alias for HDMI-A-2
    maximized: true
    minimized: false
    x: 1920
    y: 1080
    width: 3840
    height: 2110
    command: thunderbird       # from .desktop Exec= line
    single_instance: true
    tracking_uuid: a3f2b8c1-4d99-87ce-48f634523368
    _caption: 'Inbox - sam@azer.ca - Mozilla Thunderbird'   # informational
    _pid: 288264                                             # informational

  # ======
  # | Job1 |
  # ======

  - app: org.kde.konsole
    resource_class: org.kde.konsole
    desktop: Job1
    screen: main
    ...
```

Windows that appear on all virtual desktops use `desktop: '*'` and are grouped
at the end of the file under an `# All desktops` section heading.

Fields beginning with `_` are informational only and are ignored by the restore
script.  Edit the `command` field if the auto-detected command is incorrect for
any app.  The `tracking_uuid` field is assigned at capture time and re-used on
subsequent restores of the same file; do not edit it.

---

### realign

Correct window drift without relaunching any applications.  Run after
unlocking the screen saver, after extended inactivity, or whenever windows
have drifted from their intended positions.

```bash
realign [OPTIONS] [APP]
```

| Argument / Option | Default | Description |
|---|---|---|
| `APP` | | *(optional)* Realign only this app (alias or `resourceClass`). Equivalent to `--filter APP`. |
| `--setup FILE` | | Snapshot/setup YAML to read from |
| `--config FILE` | `~/.config/wayland-desktop/config.yaml` | Alternate config |
| `--all-apps` | | Process all entries, not just `on_all_desktops` ones |
| `--dry-run` | | Show what would be done without changing anything |
| `--settle-delay MS` | 400 | Milliseconds to wait after each desktop switch |
| `--filter APP` | | Only realign this app alias or `resourceClass` (repeatable) |
| `--verbose` | | Print a debug line after every KWin script call |
| `--show-window` | | Show the progress window |
| `--no-window` | | Suppress the progress window |

**How it works:**

The outer loop iterates over every virtual desktop. The inner loop processes
YAML entries that are relevant to that desktop — by default, only entries
with `desktop: '*'` (on all desktops). With `--all-apps`, single-desktop
entries are also processed when the outer loop reaches their desktop.

On each (desktop, entry) iteration: switch to the desktop, locate the window
(from session state or a live class scan), re-anchor to the correct screen,
then apply tile/maximize/minimize/raise. The originally active desktop is
restored after the loop completes.

**Examples:**

```bash
# Realign all on_all_desktops windows
realign

# Realign only chrome windows
realign chrome

# Realign all windows on all desktops
realign --all-apps

# Dry run to see what would change
realign --dry-run --verbose
```

---

### mouse-clicks

Interactive monitor for capturing mouse clicks and keyboard input as
ready-to-paste `input_actions` DSL.  Run it in a Konsole session while
interacting with your apps, then paste the output into `input_actions`
fields in your YAML setup file.

> ⚠️ **Mouse click capture works correctly.  However, replaying captured
> `{WCLICK}` and other mouse click tokens via `input_actions` is currently
> not working properly.**  Use keyboard shortcuts in `input_actions` wherever
> possible as a workaround.  See PROBLEMS.md issue 3.

```bash
mouse-clicks [OPTIONS]
# or, if the input group is not active in the session token:
sg input -c "mouse-clicks [OPTIONS]"
```

| Option | Default | Description |
|---|---|---|
| `--setup FILE` | | Snapshot YAML for window alias names |
| `--config FILE` | `~/.config/wayland-desktop/config.yaml` | Alternate config |
| `--log FILE` | | Also write output to FILE (ANSI stripped) |
| `--no-dsl` | | Print comment lines only; suppress DSL token output |
| `--verbose` | | Also log mouse-move events (very noisy) |
| `--drag-threshold N` | 5 | Pixel displacement (either axis) to distinguish a drag from a click |

**Output format:**

```
# konsole  |  "Konsole"  |  abs(6200, 1900)  |  rel(440, 100)
{WCLICK 440, 100}

# konsole  |  "Konsole"  |  abs(6200, 1900) → abs(6200, 1300)  |  rel(440, 100) → rel(440, -500)
{WDRAG 440, 100 to 440, -500}

# konsole  |  "Open File — Konsole"  |  keyboard
/home/sam/docs/{ENTER}
```

Each click produces a comment line (for context) followed by a `{WCLICK}` or
`{WRCLICK}` token line (ready to paste).  Left-button actions are logged on
release so that drags can be distinguished from clicks: movements within
`--drag-threshold` pixels are logged as `{WCLICK}`; larger movements are
logged as `{WDRAG x1,y1 to x2,y2}`.  Keyboard input accumulates silently
and is flushed as a single line when ENTER or ESC is pressed, or when the
active window changes.

**Window-relative coordinates** (`{WCLICK}` and `{WRCLICK}`) are computed
relative to the window's top-left corner at the moment of the click.  They
work for any window — including file dialogs and other tool windows opened
by the main application.

**Inline comments:** To annotate a recording session, type a `#` followed
by your comment text and press ENTER.  The comment appears in the output
and in any log file, and is ignored by the `input_actions` parser on replay:

```
# this click opens the preferences dialog{ENTER}
```

**Controls:** Ctrl-C stops the monitor and flushes any pending keyboard input.

**Requirements:** `python-evdev` (`pip install evdev --user`) and membership
in the `input` group.  See Requirements above if the group is not active in
your session token.

---

### probe

Identify the window created by a newly launched program.

```bash
probe [OPTIONS] -- <command> [args...]
```

| Option | Default | Description |
|---|---|---|
| `--timeout SECS` | 30 | Wait this long for a new window to appear |
| `--class NAME` | | Only report windows with this `resourceClass` |
| `--count N` | 1 | Stop after detecting N new windows |
| `--all` | | Collect all windows until timeout |
| `--relaunch` | | Launch even if the app is already running |
| `--output FILE` | | Write a JSON report to FILE |
| `--quiet` | | Suppress progress output |

**Examples:**

```bash
# Identify Thunderbird's window
probe -- thunderbird

# Force a new instance regardless
probe --relaunch -- thunderbird

# Save a JSON report for scripting
probe --output tb.json -- thunderbird
```

**Output example:**

```
Window: {d6a03aac-8f51-4d99-87ce-48f634523368}  +1972ms
  resourceClass    thunderbird-esr
  caption          Home - Mozilla Thunderbird
  desktopFileName  thunderbird-esr
  pid              288264
  desktop          Plumbing
  screen           HDMI-A-2
  geometry         1920,1080  3840x2110
  maximized        False

internalId handles:
    {d6a03aac-...}  [thunderbird-esr / Home - Mozilla Thunderbird]
```

The `internalId` UUID is the handle used by the library for all subsequent
operations on that window.  It changes each time the window is opened, so
always probe first rather than hard-coding UUIDs.

---

### getconfig

Inspect `~/.config/wayland-desktop/config.yaml` from the command line.  Supports
listing all properties and looking up individual values by dot-separated path.

```bash
getconfig list
getconfig lookup <key>
getconfig [--config FILE] lookup <key>
```

**Commands:**

| Command | Description |
|---|---|
| `list` | Print all top-level keys and their values |
| `lookup <key>` | Print the value at the given dot-path (no trailing newline) |

**Dot-path examples:**

```bash
getconfig lookup progress_window             # → true or false
getconfig lookup apps.brave.single_instance  # → false
getconfig lookup screens.main               # → HDMI-A-2
getconfig lookup maximize.height_tolerance  # → 50
getconfig lookup user_scripts_folder        # → ~/.config/wayland-desktop/scripts

# Shell capture (no trailing newline makes $() capture clean):
FOLDER=$(getconfig lookup user_scripts_folder)
```

Exits 0 on success, 1 if the key path is not found.  Uses the same config
file search order as `restore`: `--config FILE` if given, otherwise
`~/.config/wayland-desktop/config.yaml`.

---

### secret-service

List Secret Service entries or look up a secret by label.  Intended for use
from `restore-user-before` scripts that need passwords at restore time.

```bash
secret-service list [--verbose]
secret-service lookup <key>

# Shell capture — no trailing newline:
VPN_PASS=$(secret-service lookup "vpn-password")
```

| Command | Description |
|---|---|
| `list` | Print all item labels (and UserName attributes) from the default Secret Service collection |
| `list --verbose` | Print all attributes for each item |
| `lookup <key>` | Print the secret for the item whose label exactly matches `<key>` (no trailing newline) |

Requires KeePassXC running with Secret Service Integration enabled.


---

## Limitations

**Window resize** is not possible via the compositor on Wayland.  Only position
can be changed via `frameGeometry`; size changes require the application's
cooperation.  Use `maximize()` instead.

**Screen assignment** works by moving the window into the
target screen's coordinate space.  KWin then associates the window with that
screen.  `w.output` is read-only and `sendToOutput()` does not exist in this
KWin version.

**Flatpak applications** may not always have their `resourceClass` and `pid`
visible to KWin scripting in the same way as natively installed apps.  If
probing a Flatpak app fails, try using `--class` explicitly.

**Multi-window applications** — if an app creates multiple windows (e.g. a
splash screen followed by the main window), `WindowProbe` with `--count 1`
(the default) returns the first window that appears.  Use `--all` to capture
all windows and inspect the output to determine which is the main one.

**Session restore completeness** — snapshot and restore covers window placement
and state, not application-internal state (open tabs, loaded files, etc.).
This is a Wayland-level limitation, not specific to this library.

**Single KWinSession at a time** — the library publishes a D-Bus service name
(`org.kwinlib.Session`) and only one instance can hold that name at once.
Do not run two scripts using `KWinSession` simultaneously from the same user
session.  `WindowProbe` uses its own separate service name and can run
independently.

---

## Troubleshooting

**`Cannot connect to KWin D-Bus scripting interface`**
Ensure you are running inside an active KDE Plasma / Wayland session.  The
script must be run as the same user that owns the session.  Check that KWin is
running: `ps aux | grep kwin_wayland`.

**`Cannot publish D-Bus service ... Is another KWinSession already open?`**
A previous run crashed without cleaning up.  Wait a few seconds for the
D-Bus name to be released, or restart the session bus with
`systemctl --user restart dbus`.

**App window not detected / timeout**
Try increasing `--timeout`.  If the app is already running and you want a new
instance, add `--relaunch`.  If KWin reports a different `resourceClass` than
expected, run without `--class` to see all new windows, then add the correct
`--class` to the config.

**Snapshot shows unexpected apps**
Add the offending `resourceClass` to `system_classes` in `config.yaml`.
Running `snapshot --dry-run` is a safe way to inspect what
would be captured before committing to a file.

**Restore places window on wrong screen**
Screen connector names (`DP-4`, `HDMI-A-2`, etc.) can change after a reboot if
the GPU driver re-enumerates outputs in a different order.  Run
`snapshot --show-config` after each reboot to confirm the connector
names are stable, and update aliases in `config.yaml` if they change.

**Command not found on restore**
Edit the `command` field in the snapshot YAML file.  The auto-detected command
comes from the app's `.desktop` file `Exec=` line; if the app was not installed
via a `.desktop` file (e.g. a custom script), you will need to set this
manually.

