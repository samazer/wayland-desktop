<!-- SPDX-FileCopyrightText: 2026 Sam Azer <sam dot azer at azertech dot net>  -->
<!-- SPDX-License-Identifier: CC-BY-SA-4.0  -->
# wayland-desktop — Architecture & Design Decisions

**Project:** KDE Plasma 6 / Wayland Window Management Library  
**System:** OpenSUSE Tumbleweed, KDE Plasma 6, Wayland, Python 3.13  
**Status:** Active development  

---

## Available Documents
| Document                          | Description                                  |
|:----------------------------------|:---------------------------------------------------|
| [README.md](../README.md)         | Installation, configuration, CLI tools             |
| [OVERVIEW.md](OVERVIEW.md)        | How the library works                              |
| [INSTALL.md](INSTALL.md)          | Insetallation and setup information.               |
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

## Table of Contents

0. [Copyright &amp; License](#cl)
1. [System Context](#1-system-context)
2. [Architecture Overview](#2-architecture-overview)
3. [Module Reference](#3-module-reference)
4. [Known Limitations and Open Issues](PROBLEMS.md)
5. [Architectural Decision Records](ADRs.md)

---

### Copyright &amp; License {cl}
Wayland Desktop Restore (documentation) Copyright © 2026 by Sam Azer &lt;sam dot azer at azertech dot net&gt; is licensed under Creative Commons Attribution-ShareAlike 4.0 International. To view a copy of this license, visit https://creativecommons.org/licenses/by-sa/4.0/

#### Creative Commons Attribution-ShareAlike 4.0 International
This license requires that re-users give credit to the creator. It allows re-users to distribute, remix, adapt, and build upon the material in any medium or format, even for commercial purposes. If others remix, adapt, or build upon the material, they must license the modified material under identical terms.

- Credit must be given to the creator.
- Adaptations must be shared under the same terms.

[See the full License](https://creativecommons.org/licenses/by-sa/4.0/)

---

## 1. System Context

The goal is programmatic window management on a KDE Plasma 6 / Wayland desktop:
launching applications, identifying their windows, placing them on specific
virtual desktops and screens, and saving and restoring complete desktop layouts
across reboots.

The target system has multiple monitors, multiple virtual desktops, and a range
of applications including email clients, browsers, terminal emulators, file
managers, and productivity tools.

### Why this is non-trivial on Wayland

The X11-era toolchain (`wmctrl`, `xdotool`, `xprop`, `qdbus`) is unavailable or
non-functional on KDE Plasma 6 / Wayland:

- `wmctrl` and `xdotool` are X11-only; they have no Wayland equivalents.
- `qdbus` is a Qt5 tool removed from KDE Plasma 6 installations.
- The Wayland protocol deliberately prevents external processes from moving or
  resizing windows belonging to other applications. Only the compositor itself
  has that authority.
- `getWindowList()`, which existed in older KWin D-Bus interfaces, is absent
  from this KWin version.

The only available mechanism is KWin's own JavaScript scripting engine, which
runs inside the compositor process and is not subject to Wayland's
cross-process restrictions.

---

## 2. Architecture Overview

### Core mechanism

```
kwin_lib Python script
    │
    ├─ Publishes a D-Bus receiver service on the session bus
    │       (org.probe.KWinProbe  or  org.kwinlib.Session)
    │
    ├─ Writes a temporary .js file to /tmp
    │
    ├─ Calls /Scripting loadScript() to load it into KWin
    │
    └─ KWin executes the JS inside its QJSEngine:
           workspace.windowList()
           workspace.windowAdded  (signal)
           workspace.desktops
           workspace.screens
           w.desktops = [desktopObj]
           w.frameGeometry = {x, y, ...}
           w.setMaximize(h, v)
           w.minimized = true/false
           workspace.activeWindow = w
           w.setProperty("kwinlib_uuid", value)   ← Qt dynamic property — NOT available
                                                     in KWin JS sandbox; session state
                                                     file used instead (ADR-260321-01)
           callDBus(...)                           ← callback to the Python script
```

The library-side Python script runs a GLib main loop in a daemon thread to dispatch incoming
D-Bus calls from the script. All communication is asynchronous; the API
presents a synchronous interface using `threading.Event`.

### Data flow: window identification

```
probe.py injects KWin watcher script
    ↓
Script sends snapshot of existing windows → probe.py receives pre-launch set
Script connects workspace.windowAdded signal
    ↓
probe.py launches target application (subprocess.Popen)
    ↓
KWin fires windowAdded → script calls receiveWindowAdded() via callDBus
    ↓
probe.py filters against pre-launch snapshot → new window identified
    ↓
WindowInfo returned with internalId UUID
```

### Data flow: restore

```
Load snapshot YAML
    ↓
Load session_state.json (existing entries preserved for windows not in this run)
    ↓
For each entry:
    ignore_window=true    → subprocess.Popen (tray apps); move on immediately
    single_instance=true  → probe with relaunch=False (reuse existing)
    single_instance=false → probe with relaunch=True  (always launch)
    ↓
Window identified → place() called:
    unmaximize → move_to_desktop → move_to_screen → maximize/minimize/raise
    ↓
session_state.upsert(tracking_uuid → internalId)
    ↓
session_state.save() → ~/.config/wayland-desktop/session_state.json
```

### Session UUID tracking

Each snapshot entry carries a `tracking_uuid` (generated at capture time).
After restore, `restore` writes `~/.config/wayland-desktop/session_state.json`
mapping each `tracking_uuid` to the live `internalId` of the placed window,
along with metadata (`app_alias`, `resource_class`, `desktop`, `screen`,
`placed_at` timestamp).  A future realign utility will read this file to
re-place drifted windows on demand without re-running the full restore.

**Note:** Qt dynamic properties (`setProperty`/`property`) are not available
on window objects in the KWin JS sandbox — confirmed by `kwin_property_test.py`
on 2026-03-21.  The session state file is the adopted alternative.
See ADR-260321-01.

---

## 3. Module Reference

```
wayland-desktop/                  ← project root
│
├── restore                       Bash entry point: kdialog dialogs → restore-user-before → kwin_restore.py → restore-user-after
├── getconfig                     Wrapper: kwin_getconfig.py (list / dot-path lookup)
├── snapshot                      Wrapper: kwin_snapshot.py
├── realign                       Wrapper: kwin_realign.py
├── probe                         Wrapper: kwin_probe.py
├── mouse-clicks                  Wrapper: kwin_mouse-clicks.py
├── secret-service                Wrapper: kwin_secret_service.py (list / lookup)
├── config.yaml                   Template config file
│
└── wayland/
    ├── kwin_lib/
    │   ├── __init__.py           KWinSession context manager; launch_and_place() convenience
    │   ├── bus.py                D-Bus infrastructure: KWinBus, ScriptRunner, GLibLoop
    │   ├── click_indicator.py    Full-screen overlay for --show-clicks dot display
    │   ├── config.py             Config loading; bidirectional alias resolution
    │   ├── desktop.py            DesktopManager: enumerate desktops and screens;
    │   │                         switch_to_desktop(); active_window_and_cursor()
    │   ├── input_injection.py    XDG RemoteDesktop portal session; DSL parser and executor
    │   │                         (keystrokes, mouse clicks, drags, scroll, sleep, {PASS})
    │   ├── models.py             WindowInfo, DesktopInfo, ScreenInfo dataclasses
    │   ├── probe.py              WindowProbe: launch app and identify new window
    │   ├── progress_window.py    PyQt6 progress window (spawned child process)
    │   ├── secret_service.py     Secret Service (so far tested only on KeePassXC) client for {PASS key} tokens
    │   ├── session_state.py      Session state file: tracking_uuid → internalId mappings
    │   ├── snapshot.py           Layout capture; .desktop file lookup; YAML serialisation
    │   └── window.py             WindowManager: all window mutation operations;
    │                             ensure_maximized()
    │
    ├── getconfig         CLI: config.yaml inspection (list and dot-path lookup)
    ├── mouse-clicks      CLI: capture mouse clicks, drags, and keyboard as DSL
    ├── probe             CLI: identify windows
    ├── realign           CLI: drift correction — re-tile without relaunching
    ├── restore           CLI: restore layout from YAML (five phases)
    ├── secret-service    CLI: Secret Service list and lookup utility
    └── snapshot          CLI: capture current layout to YAML
```

### Config file

`~/.config/wayland-desktop/config.yaml` — aliases for desktops, screens, and apps.
Aliases are bidirectional and fall-through: unknown names pass through
unchanged. The special values `"*"` and `"all"` are built-in sentinels meaning
"all virtual desktops".

### Snapshot YAML format

Grouped by desktop in config order. All-desktops windows last. Per-entry:

```yaml
- app: thunderbird          # alias from config, or resourceClass
  resource_class: thunderbird-esr
  desktop_filename: thunderbird-esr
  desktop: Plumbing         # alias or '*'
  screen: main              # alias or connector name
  maximized: true
  minimized: false
  x: 1920
  y: 1080
  width: 3840
  height: 2110
  command: thunderbird      # from .desktop Exec= line
  single_instance: true
  tracking_uuid: a3f2b8c1-4d99-87ce-...
  _caption: 'Inbox - Mozilla Thunderbird'   # informational
  _pid: 288264                              # informational
```

---

## 4. Known Limitations and Open Issues

See **[PROBLEMS.md](PROBLEMS.md)** for the full list of known limitations,
confirmed bugs, and unimplemented features.
