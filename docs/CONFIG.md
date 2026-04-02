<!-- SPDX-FileCopyrightText: 2026 Sam Azer <sam dot azer at azertech dot net>  -->
<!-- SPDX-License-Identifier: CC-BY-SA-4.0  -->
# wayland-desktop — Configuration Reference

**File:** `~/.config/wayland-desktop/config.yaml`

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

- [Copyright &amp; License](#copyright--license)
- [Overview](#overview)
- [desktops — virtual desktop aliases](#desktops--virtual-desktop-aliases)
- [screens — monitor aliases](#screens--monitor-aliases)
- [apps — application overrides](#apps--application-overrides)
- [snapshot — snapshot settings](#snapshot--snapshot-settings)
- [default_setup — default snapshot file](#default_setup--default-snapshot-file)
- [progress_window — progress window default](#progress_window--progress-window-default)
- [maximize — geometry tolerances](#maximize--geometry-tolerances)
- [use_secret_service — Secret Service integration](#use_secret_service--secret-service-integration)
- [user_scripts_folder — user script directory](#user_scripts_folder--user-script-directory)
- [Full annotated example](#full-annotated-example)

---

### Copyright &amp; License
Wayland Desktop Restore (documentation) Copyright © 2026 by Sam Azer &lt;sam dot azer at azertech dot net&gt; is licensed under Creative Commons Attribution-ShareAlike 4.0 International. To view a copy of this license, visit https://creativecommons.org/licenses/by-sa/4.0/

#### Creative Commons Attribution-ShareAlike 4.0 International
This license requires that re-users give credit to the creator. It allows re-users to distribute, remix, adapt, and build upon the material in any medium or format, even for commercial purposes. If others remix, adapt, or build upon the material, they must license the modified material under identical terms.

- Credit must be given to the creator.
- Adaptations must be shared under the same terms.

[See the full License](https://creativecommons.org/licenses/by-sa/4.0/)

---

## Overview

`config.yaml` is the configuration file for all wayland-desktop
scripts.  It defines aliases for virtual desktops and monitors,
application-specific overrides, snapshot exclusion rules, and script
defaults.

Note that Desktop and Screen aliases are **bidirectional**: once
defined, either the alias or the canonical KWin name is accepted in
any script argument or snapshot field.  Unknown names pass through
unchanged without error.

This file must be located at `~/.config/wayland-desktop/config.yaml`.
If not found, the desired folder can be passed through the command
line.  Otherwise, if no configuration is found, the scripts will use
built-in defaults (no aliases, no overrides).

---

## desktops — virtual desktop aliases

Maps short user-friendly names to the exact desktop names shown in KWin's
Virtual Desktop settings.

```yaml
desktops:
  Main:     "Main"        # identity alias — explicit for clarity
  Job1:     "Job 1"
  Job2:     "Job 2"
  Media:    "Media"
```

**Key:** alias (the name you use on the command line and in snapshot files)  
**Value:** the exact KWin virtual desktop name (case-sensitive)

Identity aliases (where alias equals value) are harmless and useful for
documentation — they make the mapping explicit even when no translation
is needed.

---

## screens — monitor aliases

Maps short user-friendly names to KWin connector names.

```yaml
screens:
  top:         "DP-4"       # LG FULL HD, 2885,0  1920x1080
  main:        "HDMI-A-2"   # primary large display, 3840x2160
  right:       "DP-5"       # Acer B326HUL, 5760,1800  2560x1440
  left-bottom: "Unknown-2"  # Acer EK271, 0,2160  1920x1080
  left-top:    "HDMI-A-1"   # ASUS VG259Q3A, 0,1080  1920x1080
```

**Key:** alias  
**Value:** KWin connector name (e.g. `DP-4`, `HDMI-A-2`)

To find the connector name for each monitor, run `probe` or
inspect the output of `restore --verbose` — the `screen_name`
field in `get_screen_geometry` results shows the live connector name.

---

## apps — application overrides

Per-application configuration keyed by alias.

```yaml
apps:
  thunderbird:
    resourceClass:   "thunderbird-esr"
    single_instance: true
    tile:            ""

  konsole:
    resourceClass:   "org.kde.konsole"
    single_instance: false
    tile:            "right"

  keepassxc:
    resourceClass:   "org.keepassxc.KeePassXC"
    single_instance: true
    ignore_window:   true
```

### Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `resourceClass` | string | — | The KWin `resourceClass` identifier for this app. Run `probe` to find it. Required if the alias differs from the resourceClass. |
| `single_instance` | bool | `false` | If `true`, skip launching if a window with this resourceClass is already open. If `false`, always launch a new instance. |
| `tile` | string | `""` | Default tile position applied after placement. Accepted values: `left`, `right`, `top`, `bottom`, `top-left`, `top-right`, `bottom-left`, `bottom-right`. Empty string means no tiling (window is placed as-is or maximized). |
| `command` | string | — | Override the launch command derived from the `.desktop` file. Use only when the `.desktop`-derived command is incorrect. |
| `ignore_window` | bool | `false` | If `true`, launch the app without waiting for a window to appear. Use for tray-only apps (e.g. KeePassXC) that launch directly to the system tray. |

### Notes

- The `tile` field here acts as a **default**.  A `tile:` field in a snapshot
  entry overrides it for that specific window.
- `resourceClass` values are case-sensitive.  Use `probe` or
  `restore --verbose` to confirm the exact value KWin reports.
- Apps not listed in `apps:` can still appear in snapshots — they will be
  launched using the command from their `.desktop` file and no overrides apply.

---

## snapshot — snapshot settings

Controls `snapshot` behaviour.

```yaml
snapshot:
  system_classes:
    - plasmashell
    - ibus-ui-gtk3
    - kwin_wayland
    - krunner
    - ksmserver
    - kded6
    - polkit-kde-authentication-agent-1
    - xwaylandvideobridge
    - org.kde.plasmashell
    - latte-dock
    - plank

  snapshot_dir: "~/.config/wayland-desktop/snapshots"
```

### Fields

| Field | Type | Description |
|---|---|---|
| `system_classes` | list of strings | Window `resourceClass` values to exclude from snapshots. These are compositor and system windows with no meaning outside the current session. |
| `snapshot_dir` | path | Directory where snapshot files are saved by `snapshot`. `~` is expanded. |

### Extending system_classes

Add any application whose windows should never appear in snapshots.  Common
additions: `yakuake`, `albert`, `ulauncher`, `gnome-pie`.

---

## default_setup — default snapshot file

```yaml
default_setup: "desktop-setup.yaml"
```

The name of the snapshot file `restore` uses when `--setup` is not
given on the command line.  Leave blank (`""`) to require `--setup` explicitly.

This is a **bare filename** (not a path).  `restore` searches for it
in order:

1. The current working directory
2. `~/.config/wayland-desktop/`
3. `snapshot_dir` (defined in the `snapshot:` section above)

If not found in any location, the script exits with an informative error.

---

## progress_window — progress window default

```yaml
progress_window: ""
```

Controls whether `restore` shows a PyQt6 progress window, and which screen
to place it on.

| Value | Behaviour |
|---|---|
| `""` (blank, default) | No progress window unless `--show-window` is passed |
| `screen-alias` | Show the window and move it to the named screen |

When a screen alias is given the following additional behaviour applies:

- The window is moved to the named screen immediately after it opens.
- `keepBelow` is set so the window stays behind other windows during
  phases 1–3 and phase 4.
- The window is minimized entering Phase 5 (desktop switching).
- At the end of Phase 5 `keepBelow` is cleared, the window is unminimized,
  and it is raised to the front so the completion summary is visible.

The alias must be one of the keys defined in the `screens:` section of
`config.yaml`.  `kwin_restore.py` exits with an error if the alias is not
found.

**Command-line overrides:**

- `--show-window` — forces the window to appear for this run.  If
  `progress_window` contains a valid screen alias, screen placement and
  `keepBelow` still apply.  If `progress_window` is blank, the window
  appears wherever KWin places it with no `keepBelow`.
- `--no-window` — suppresses the window for this run regardless of the
  config setting.

---

## maximize — geometry tolerances

```yaml
maximize:
  width_tolerance:  0
  height_tolerance: 50
```

These values are used by `ensure_maximized()` in `kwin_lib/window.py` to
verify that a window visually fills its screen after `maximize()` is called.

On some screens (particularly high-DPI displays) KWin sets the maximized
flag but the Wayland client does not immediately resize to fill the screen.
`ensure_maximized()` reads back the window and screen geometry after
`maximize()` and retries (`unmaximize()` + `maximize()`) if the dimensions
are not within tolerance.

**Note:** Due to a KWin JS sandbox limitation (`frameGeometry` does not
update within the same script execution after `setMaximize()`), the
geometry read-back is currently unreliable.  `ensure_maximized()` is
implemented and called but may not be effective in all cases.  See
PROBLEMS.md issue 6 for details and future investigation notes.

| Field | Type | Default | Description |
|---|---|---|---|
| `width_tolerance` | int | `0` | Pixels by which window width may fall short of screen width and still be considered maximized. |
| `height_tolerance` | int | `50` | Pixels by which window height may fall short of screen height. Set to the height of a docked taskbar panel (the default of 50 covers a standard Plasma panel at the top or bottom of the screen). If your taskbar is on the left or right side, set this to `0` and set `width_tolerance` to the panel width instead. |

---

## use_secret_service — Secret Service integration

```yaml
use_secret_service: false
```

When `true`, `restore` opens a connection to the Secret Service
(implemented by KeePassXC) before Phase 1, enabling the `{PASS key}`
input_actions DSL token.  The connection is primed at startup — each required
password is fetched once and discarded — so that any KeePassXC approval dialogs
appear at a known, calm moment before windows are launched or moved.

| Value | Behaviour |
|---|---|
| `false` (default) | No Secret Service connection is attempted; `{PASS}` tokens fail with a clear error |
| `true` | A Secret Service session is opened before Phase 1 and closed after Phase 4 |

**Prerequisites:**

1. KeePassXC must be running and have its database unlocked before the restore
   runs.
2. Secret Service Integration must be enabled in KeePassXC:
   **Tools → Settings → Secret Service Integration → Enable KeePassXC Secret
   Service Integration**.
3. The database must contain entries whose **title** (label) matches the keys
   used in `{PASS key}` tokens.

**Failure behaviour:** If the Secret Service cannot be reached (KeePassXC not
running, or not unlocked), a warning is logged and `{PASS}` tokens for that
run are skipped as non-fatal errors.  All other `input_actions` tokens continue
to execute normally.

**Security note:** Passwords are fetched at restore time and held in memory
only for the duration of the `send_char()` loop that types them.  They are
never written to disk, never logged, and are not stored in the snapshot YAML
file.  See also `docs/PROBLEMS.md` (issue 4 — now resolved).

---

## user_scripts_folder — user script directory

```yaml
user_scripts_folder: "~/.config/wayland-desktop/scripts"
```

Directory where the `restore` wrapper looks for user hook scripts.
Relative paths are resolved relative to the wayland-desktop project root (the
directory containing `restore`).  `~` is expanded.

| Value | Meaning |
|---|---|
| `"~/.config/wayland-desktop/scripts"` (default) | Standard config directory sub-folder |
| `"~/scripts/wayland-desktop"` | Another home-relative path |
| `"/opt/restore-scripts"` | Absolute path |
| `"./"` | The project root (same directory as `restore`) |

The `restore` script reads this value via `getconfig lookup user_scripts_folder`
before each run.  If `getconfig` fails for any reason (e.g. no config file),
`restore` falls back to `"./"`.

Two optional hook scripts are looked up in this folder:

**`restore-user-before`** — run after the startup dialogs but before
`kwin_restore.py` launches.  Typical use: start background services (VPNs,
SSH agents, KeePassXC) that the restore depends on.  The `getconfig` and
`secret-service` wrappers are available inside this script.

**`restore-user-after`** — run after `kwin_restore.py` returns, regardless of
whether it succeeded or failed.  The exit code of `kwin_restore.py` is
preserved and re-applied after this hook returns, so it cannot mask a restore
failure.  Typical use: post-restore tasks such as opening a terminal, sending a
notification, or cleaning up temporary state set up by `restore-user-before`.

---

## Full annotated example

```yaml
# =============================================================================
# ~/.config/wayland-desktop/config.yaml
# =============================================================================

# Virtual desktop aliases
# Format:  alias: "Canonical KWin Desktop Name"
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

# Monitor aliases
# Format:  alias: "KWin connector name"
screens:
  top:          "DP-4"       # LG FULL HD,      2885,0     1920x1080
  main:         "HDMI-A-2"   # FireTV/primary,  1920,0     3840x2160
  right:        "DP-5"       # Acer B326HUL,    5760,1800  2560x1440
  left-bottom:  "Unknown-2"  # Acer EK271,      0,2160     1920x1080
  left-top:     "HDMI-A-1"   # ASUS VG259Q3A,   0,1080     1920x1080

# Application overrides
apps:
  thunderbird:
    resourceClass:   "thunderbird-esr"
    single_instance: true
    tile:            ""          # maximized — no tile

  brave:
    resourceClass:   "brave-browser"
    single_instance: false
    tile:            ""

  konsole:
    resourceClass:   "org.kde.konsole"
    single_instance: false
    tile:            ""

  dolphin:
    resourceClass:   "org.kde.dolphin"
    single_instance: false
    tile:            ""

  obsidian:
    resourceClass:   "obsidian"
    single_instance: true
    tile:            ""

  keepassxc:
    resourceClass:   "org.keepassxc.KeePassXC"
    single_instance: true
    tile:            ""
    ignore_window:   true       # launches to tray — no window expected

# Snapshot settings
snapshot:
  system_classes:
    - plasmashell
    - ibus-ui-gtk3
    - kwin_wayland
    - krunner
    - ksmserver
    - kded6
    - polkit-kde-authentication-agent-1
    - xwaylandvideobridge
    - org.kde.plasmashell
    - latte-dock
    - plank

  snapshot_dir: "~/.config/wayland-desktop/snapshots"

# Default snapshot file for restore (bare filename, not a path)
default_setup: "desktop-setup.yaml"

# Show progress window on the left-top screen during restore.
# Leave blank ("") to disable the progress window by default.
progress_window: ""

# Enable Secret Service integration for {PASS key} tokens in input_actions.
# Requires KeePassXC running with Secret Service Integration enabled.
use_secret_service: false

# Directory where the `restore` wrapper looks for restore-user-before and
# restore-user-after hook scripts.
# Default: ~/.config/wayland-desktop/scripts
user_scripts_folder: "~/.config/wayland-desktop/scripts"

# Maximize geometry tolerances (see ensure_maximized in kwin_lib/window.py)
maximize:
  width_tolerance:  0
  height_tolerance: 50
```
