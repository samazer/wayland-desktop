<!-- SPDX-FileCopyrightText: 2026 Sam Azer <sam dot azer at azertech dot net>  -->
<!-- SPDX-License-Identifier: CC-BY-SA-4.0  -->
# wayland-desktop — INSTALLATION

(This document is a work-in-progress.)

---

## Available Documents
| Document                          | Description                                  |
|:----------------------------------|:---------------------------------------------------|
| [README.md](../README.md)         | Installation, configuration, CLI tools             |
| [OVERVIEW.md](OVERVIEW.md)        | How the library works                              |
| [INSTALL.md](INSTALL.md)          | Installation and setup information.               |
| [ARCHITECTURE.md](ARCHITECTURE.md) | System context and design overview                 |
| [PROBLEMS.md](PROBLEMS.md)        | Known limitations and open issues                  |
| [../WARNINGS.md](../WARNINGS.md) | User-Level Usage Warnings | 
| [GLOSSARY.md](GLOSSARY.md)        | Definitions of terms                               |
| [ADRs.md](ADRs.md)                | Architectural Decision Records                     |
| [REQUIREMENTS.md](REQUIREMENTS.md) | Project requirements and implementation status     |
| [DATA-MODELS.md](DATA-MODELS.md)  | WindowInfo, DesktopInfo, ScreenInfo dataclasses    |
| [LIBRARY-API.md](LIBRARY-API.md)  | Python API reference                               |
| [CONFIG.md](CONFIG.md)            | Configuration file for all wayland-desktop scripts |
| [SNAPSHOT.md](SNAPSHOT.md)        | Snapshot file format and input_actions DSL         |
| [../ABOUT.md](../ABOUT.md)         | Some background information about this project |
| [dev/README.md](dev/README.md)    | index and annotated notes for all dev scripts      |

---

## Contents

0. [Copyright &amp; License](#copyright--license)
1. [Requirements](#requirements)
2. [Installation](#installation)
3. [Configuration](#configuration)
4. [File layout](#file-layout)

---

### Copyright &amp; License
Wayland Desktop Restore (documentation) Copyright © 2026 by Sam Azer &lt;sam dot azer at azertech dot net&gt; is licensed under Creative Commons Attribution-ShareAlike 4.0 International. To view a copy of this license, visit https://creativecommons.org/licenses/by-sa/4.0/

#### Creative Commons Attribution-ShareAlike 4.0 International
This license requires that re-users give credit to the creator. It allows re-users to distribute, remix, adapt, and build upon the material in any medium or format, even for commercial purposes. If others remix, adapt, or build upon the material, they must license the modified material under identical terms.

- Credit must be given to the creator.
- Adaptations must be shared under the same terms.

[See the full License](https://creativecommons.org/licenses/by-sa/4.0/)

---

## Requirements

| Package | Purpose | Install |
|---|---|---|
| `python313-pydbus` | D-Bus communication with KWin | `sudo zypper install python313-pydbus` |
| `python313-gobject` | GLib main loop for D-Bus callbacks | `sudo zypper install python313-gobject` |
| `python313-PyYAML` | Config and snapshot files | `sudo zypper install python313-PyYAML` |
| `python313-qt6` / `python313-PyQt6` | Progress window and click indicator overlay | `sudo zypper install python313-qt6` |
| `python-evdev` | Mouse and keyboard monitoring (`mouse-clicks` only) | `pip install evdev --user` |

The first four are standard packages in the OpenSUSE Tumbleweed repositories.
PyQt6 is required for the progress window and the click indicator overlay; all
other scripts work without it if `--no-window` is used or `progress_window` is
`false` in `config.yaml`.  `python-evdev` is only required for `mouse-clicks`.

`mouse-clicks` also requires the user to be in the `input` group to
read from `/dev/input/event*`.  On some systems (notably with SDDM on
Tumbleweed) the group may be present in `/etc/group` but not active in the
login session token even after a reboot.  In that case use:

```bash
sg input -c "mouse-clicks [OPTIONS]"
```

---

## Installation

No installation step is required.  Copy the `wayland-desktop/` directory to
any convenient location on your filesystem.

```
wayland-desktop/
├── restore        ← run to restore your desktop setup
├── snapshot       ← run this to restore your desktop
├── realign        ← run this if your windows bounce around
├── probe          ← tool to query a window
├── mouse-clicks   ← tool to record keystrokes and mouse clicks
├── getconfig      ← scriptable access to configuration file properties
├── secret-service ← scriptable access to Secret Service passwords
├── config.yaml    ← template; copy to ~/.config/wayland-desktop/
│
└── wayland/
    ├── kwin_lib/
    │   ├── __init__.py
    │   ├── bus.py
    │   ├── click_indicator.py
    │   ├── config.py
    │   ├── desktop.py
    │   ├── input_injection.py
    │   ├── models.py
    │   ├── probe.py
    │   ├── progress_window.py
    │   ├── secret_service.py
    │   ├── session_state.py
    │   ├── snapshot.py
    │   └── window.py
    ├── getconfig
    ├── mouse-clicks
    ├── probe
    ├── realign
    ├── restore
    ├── secret-service
    └── snapshot
```

> **Note:** The bash wrapper scripts (`restore`, `snapshot`, etc.) locate the
> `wayland/` directory relative to their own path, so they work correctly
> regardless of where you run them from.  Whether scripts in `wayland/` can be
> invoked directly from arbitrary working directories is tracked as an open
> issue — see PROBLEMS.md.

---

## Configuration

The config file lives at `~/.config/wayland-desktop/config.yaml`.  Create it once;
all scripts read it automatically on startup.

A template with your desktop and monitor names pre-filled is provided as
`config.yaml` alongside this documentation.  Copy it into place:

```bash
mkdir -p ~/.config/wayland-desktop
cp config.yaml ~/.config/wayland-desktop/config.yaml
```

The config has three sections.

### Desktop aliases

Map short alias names to the full KWin virtual desktop names.

```yaml
desktops:
  Main:     "Main"
  Job1:     "Job 1"
  Job2:     "Job 2"
  Job3:     "Job 3"
  Plumbing: "Plumbing"
  Writing:  "Writing"
  Task1:    "Task 1"
  Task2:    "Task 2"
  Task3:    "Task 3"
  Media:    "Media"
```

Aliases are bidirectional.  Once `Job1` is defined as an alias for `"Job 1"`,
you can use either `Job1` or `Job 1` everywhere — on the command line, in
snapshot files, and in the library API.  Unknown names always pass through
unchanged, so you can use canonical KWin names directly without defining an
alias.

**All-desktops sentinel:** the special values `*` and `all` (case-insensitive)
are recognised everywhere a desktop name is accepted.  They cause the window to
be made visible on all virtual desktops simultaneously, rather than assigned to
one specific desktop.  No alias definition is required — these are built-in.

### Screen aliases

Map friendly names to the KWin connector names reported by your graphics
drivers.

```yaml
screens:
  top:          "DP-4"
  main:         "HDMI-A-2"
  right:        "DP-5"
  left-bottom:  "Unknown-2"
  left-top:     "HDMI-A-1"
```

Connector names for your system can be found by running:

```bash
snapshot --show-config
```

or by running `snapshot --dry-run` and reading the `screen` field in
the output.

### App overrides

Override per-app settings.  The key is your chosen alias for the app.

```yaml
apps:
  thunderbird:
    resourceClass:   "thunderbird-esr"   # KWin's resourceClass for this app
    single_instance: true                # never launch a second copy

  konsole:
    resourceClass:   "org.kde.konsole"
    single_instance: false               # always launch fresh instance

  dolphin:
    resourceClass:   "org.kde.dolphin"
    single_instance: false

  obsidian:
    resourceClass:   "obsidian"
    single_instance: true
```

`single_instance: true` means the restore script will reuse an existing window
if the app is already open, rather than launching a new one.  Use `true` for
apps like Thunderbird that should only ever have one instance.  Use `false` for
terminal emulators and file managers that you want running on multiple desktops
simultaneously.

`resourceClass` is the identifier KWin uses internally.  If you are unsure
what it is for a given app, run `probe -- appname` and look at the
`resourceClass` field in the output.

### Snapshot settings

```yaml
snapshot:
  system_classes:
    - plasmashell
    - ibus-ui-gtk3
    - kwin_wayland
    - krunner
    # ... add more as needed

  snapshot_dir: "~/.config/wayland-desktop/snapshots"
```

`system_classes` lists the KWin `resourceClass` values for compositor and
desktop shell windows that have no meaning outside the current session.  These
are excluded from snapshots automatically.  The default list covers the most
common Plasma system windows; extend it if you see noise in your snapshots.

### Default setup file

```yaml
default_setup: "setup.yaml"
```

`default_setup` names the snapshot file that `restore` uses when no `--setup`
argument is given.  Setting it once means you can run the restore with no
arguments at all:

```bash
restore              # restores default_setup
restore chrome       # restores only chrome from default_setup
```

The value is a bare filename, not a full path.  The restore script searches for
it in order: the current working directory, `~/.config/wayland-desktop/`, then
`snapshot_dir`.  Leave it blank or omit the key to require `--setup` on every
invocation.

### Maximize geometry tolerances

```yaml
maximize:
  width_tolerance:  0
  height_tolerance: 50
```

Controls how `ensure_maximized()` decides whether a window has successfully
filled the screen after `maximize()` is called.  The `height_tolerance` default
of 50 pixels accounts for a standard Plasma taskbar panel docked at the top or
bottom of the screen.  Adjust if your panel is a different height or is docked
to a side edge.  See `docs/CONFIG.md` for full details.

---

## File layout

```
restore                 Restore the desktop from the specified setup file
getconfig               Scriptable tool to return the values of properties in 
                        the config.yaml file. Can also list all the properties.
mouse-clicks            Tool to log all keystrokes and mouse clicks to help 
                        prepare the input_actions scripts.
probe                   Tool to query a window
realign                 Tool to move windows back to where they are supposed to
                        be (in case windows bounce around, for example, after
						logging back into the system through the screen saver.)
secret-service          Scriptable tool to retrieve passwords from the Secret 
                        Service.  Can also list all the available tokens.
snapshot                Tool to scan the current desktop and save a Desktop 
                        Setup (snapshot) file.
restore-user-before     (optional) User-supplied script run by `restore` before 
                        kwin_restore.py launches.  Use this script, for example,
						to open a Secret Service tool (such as KeepassXC) and 
						unlock encrypted folders.  The wayland-desktop scripts
						will look at the user_scripts_folder property in the
						config.yaml file to find the user scripts.  The default
						location is ~/.config/wayland-desktop/scripts.
restore-user-after      (optional) User-supplied script run by `restore` after
                        kwin_restore.py returns, regardless of success or failure.
						Can be used for any cleanup actions that might be needed.
                        The wayland-desktop scripts will look at the 
						user_scripts_folder property in the config.yaml file to 
						find this script, as above.


wayland/
│
├── kwin_lib/
│   ├── __init__.py         Public API, KWinSession context manager,
│   │                       launch_and_place() convenience function
│   ├── bus.py              D-Bus infrastructure: KWinBus, ScriptRunner,
│   │                       CallbackReceiver, GLibLoop
│   ├── click_indicator.py  Full-screen transparent overlay that draws a
│   │                       brief red dot at each injected pointer position
│   │                       during phase 4 (activated by --show-clicks)
│   ├── config.py           Config loading, bidirectional alias resolution,
│   │                       maximize tolerance fields
│   ├── desktop.py          DesktopManager: list/find virtual desktops and screens,
│   │                       switch_to_desktop(), active_window_and_cursor()
│   ├── input_injection.py  XDG RemoteDesktop portal session, DSL parser and
│   │                       executor (keystrokes, mouse, scroll, sleep)
│   ├── models.py           WindowInfo, DesktopInfo, ScreenInfo dataclasses
│   ├── probe.py            WindowProbe: launch app and identify its window
│   ├── progress_window.py  PyQt6 progress window (spawned child process)
│   ├── secret_service.py   Secret Service (so far tested only on KeePassXC) integration for
│   │                       {PASS key} DSL tokens
│   ├── session_state.py    Session state: tracking_uuid → internalId mappings
│   ├── snapshot.py         Layout capture, .desktop file lookup, YAML serialisation
│   └── window.py           WindowManager: move, maximize, minimize, place,
│                           tile, ensure_maximized()
│
├── getconfig       CLI: config inspection (list and dot-path lookup)
├── mouse-clicks    CLI: capture mouse clicks and keyboard as DSL
├── probe           CLI wrapper for WindowProbe
├── realign         CLI: drift correction — re-tile without relaunching
├── restore         CLI: restore layout from YAML snapshot (5 phases)
├── secret-service  CLI: Secret Service list and lookup utility
└── snapshot        CLI: capture current layout to YAML

~/.config/wayland-desktop/
├── config.yaml                  User aliases, app overrides, tolerances
├── session_state.json           Session state: tracking_uuid → internalId
├── portal_session_token         XDG RemoteDesktop portal restore token
├── progress_window.json         Progress window saved geometry
└── snapshots/                   Auto-named snapshot files (YYYY-MM-DD_HHMM.yaml)
```
