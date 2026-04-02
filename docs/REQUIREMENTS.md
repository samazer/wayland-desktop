<!-- SPDX-FileCopyrightText: 2026 Sam Azer <sam dot azer at azertech dot net>  -->
<!-- SPDX-License-Identifier: CC-BY-SA-4.0  -->
# wayland-desktop — Requirements

**Project:** KDE Plasma 6 / Wayland Window Management Library  
**System:** OpenSUSE Tumbleweed, KDE Plasma 6, Wayland  
**Status:** Active development  

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
- [Background and motivation](#background-and-motivation)
- [System description](#system-description)
- [Functional requirements](#functional-requirements)
  - [REQ-01 Window identification](#req-01-window-identification)
  - [REQ-02 Window placement](#req-02-window-placement)
  - [REQ-03 Window state control](#req-03-window-state-control)
  - [REQ-04 Configuration and aliases](#req-04-configuration-and-aliases)
  - [REQ-05 Layout snapshot](#req-05-layout-snapshot)
  - [REQ-06 Layout restore](#req-06-layout-restore)
  - [REQ-07 Session tracking](#req-07-session-tracking)
  - [REQ-08 Window drift correction](#req-08-window-drift-correction)
  - [REQ-09 Window tiling and keystroke control](#req-09-window-tiling-and-keystroke-control)
  - [REQ-10 Application content setup](#req-10-application-content-setup)
- [Non-functional requirements](#non-functional-requirements)
- [Constraints](#constraints)
- [Out of scope](#out-of-scope)
- [Implementation status](#implementation-status)

---

### Copyright &amp; License
Wayland Desktop Restore (documentation) Copyright © 2026 by Sam Azer &lt;sam dot azer at azertech dot net&gt; is licensed under Creative Commons Attribution-ShareAlike 4.0 International. To view a copy of this license, visit https://creativecommons.org/licenses/by-sa/4.0/

#### Creative Commons Attribution-ShareAlike 4.0 International
This license requires that re-users give credit to the creator. It allows re-users to distribute, remix, adapt, and build upon the material in any medium or format, even for commercial purposes. If others remix, adapt, or build upon the material, they must license the modified material under identical terms.

- Credit must be given to the creator.
- Adaptations must be shared under the same terms.

[See the full License](https://creativecommons.org/licenses/by-sa/4.0/)

---

## Background and motivation

KDE Plasma on X11 provided session restore functionality: on login, all
previously open applications were re-launched and their windows were
restored to the correct positions, virtual desktops, and monitors.  This
relied on X11-era mechanisms (`_NET_WM_*` window properties, `wmctrl`,
`xdotool`, XSMP session management) that have no direct equivalents in
the Wayland protocol.

When the desktop was migrated to KDE Plasma 6 on Wayland, session restore
ceased to function reliably.  KWin bug #15329 (no ability to save and
restore positions of native Wayland windows) and bug #436318 (no ability
to restore window content and state) remain open as of early 2026.
KDE Plasma 6.8 (expected October 2026) will be Wayland-exclusive, removing
X11 as a fallback entirely.

The session restore gap is particularly significant for a multi-monitor,
multi-desktop power user workflow where re-establishing the workspace after
a reboot manually is time-consuming and error-prone.

This library is a user-space workaround for the missing session restore,
built on KWin's own JavaScript scripting engine — the only mechanism
available to a Wayland compositor that is not subject to the cross-process
restrictions of the Wayland protocol.

The longer-term ambition extends beyond simple window placement to full
workspace reconstruction: launching applications, setting up terminal tab
layouts, loading browser sessions, and running initial commands — everything
needed to return the desktop to a productive state after a reboot.

---

## System description

Hardware includes multiple monitors and graphics cards running multiple
virtual desktops, with common applications such as Thunderbird and Brave
Browser alongside terminal emulators, file managers, and productivity tools.
Some applications run as single instances; others run simultaneously across
multiple desktops.

**Software:**
- OpenSUSE Tumbleweed, KDE Plasma 6, Wayland, Python 3.13

---

## Functional requirements

### REQ-01 Window identification

**REQ-01.1** — The system shall be capable of launching an application and
identifying the window it creates, returning a stable handle for that window.

**REQ-01.2** — The system shall detect when an application is already running
and return the handle of the existing window without launching a second instance,
unless explicitly instructed to do otherwise.

**REQ-01.3** — The system shall assign a persistent tracking UUID to each managed
window at restore time, enabling the window to be found again later within the
same session by UUID scan rather than by title or class matching alone.

**REQ-01.4** — Window identification shall work correctly for applications that
take varying amounts of time to display their main window (fast and slow starters).

**REQ-01.5** — The system shall correctly distinguish between splash screens,
transient dialogs, and the application's main window.

**REQ-01.6** — Some applications (e.g. KeePassXC) launch directly to the system
tray and never display a main window.  The system shall support an `ignore_window`
flag to launch such applications without waiting for a window to appear.

---

### REQ-02 Window placement

**REQ-02.1** — The system shall be capable of moving a window to a specified
virtual desktop, identified by name or 1-based number.

**REQ-02.2** — The system shall be capable of moving a window to a specified
monitor, identified by a user-defined alias or the KWin connector name
(e.g. `HDMI-A-2`).

**REQ-02.3** — The system shall support placing a window on all virtual desktops
simultaneously, using the sentinel value `*` or `all`.

**REQ-02.4** — When moving a window to a specific monitor, the window shall be
correctly associated with that monitor before any subsequent state operations
(e.g. maximize) are applied.  This is achieved by the three-phase restore
design: moves happen in phase 2 and are followed by a settle delay before
phase 3 applies state operations.

**REQ-02.5** — The system shall support moving a window to an absolute position
within the compositor coordinate space.

---

### REQ-03 Window state control

**REQ-03.1** — The system shall support maximizing a window on its current or
target monitor.

**REQ-03.2** — The system shall support minimizing a window to the taskbar.

**REQ-03.3** — The system shall support restoring a window from maximized or
minimized state.

**REQ-03.4** — The system shall support raising and focusing a window.

**REQ-03.5** — The system shall support snapping a window to predefined tile
positions (left half, right half, top half, bottom half, quadrants) using
KWin's built-in tiling operations.

**REQ-03.6** — The system shall note that window resize via the compositor is
not possible on Wayland.  Applications control their own size; only maximize
and tile operations are reliable for size management.

---

### REQ-04 Configuration and aliases

**REQ-04.1** — The system shall support user-defined aliases for virtual desktop
names (e.g. `Job1` → `Job 1`).

**REQ-04.2** — The system shall support user-defined aliases for monitor names
(e.g. `main` → `HDMI-A-2`).

**REQ-04.3** — The system shall support user-defined aliases for applications,
mapping a friendly name to the KWin `resourceClass` string
(e.g. `thunderbird` → `thunderbird-esr`).

**REQ-04.4** — All aliases shall be bidirectional: either the alias or the
canonical KWin name shall be accepted in any context.  Unknown names shall
pass through unchanged without error.

**REQ-04.5** — The configuration shall specify, per application, whether it is
a single-instance application (only one window should ever be open) or
a multi-instance application (multiple windows may be open simultaneously
across different desktops).

**REQ-04.6** — Configuration shall be stored in a single human-readable YAML
file at `~/.config/wayland-desktop/config.yaml`.

**REQ-04.7** — The configuration shall support a `default_setup` property
naming a snapshot file to be used by default by `restore`.  When
`default_setup` is set, `restore` may be invoked without any explicit
`--setup` argument.  When it is blank or absent, `--setup` is required.
The `default_setup` value is treated as a bare filename (not a path) and
searched in order: the current working directory, `~/.config/wayland-desktop/`, then
`snapshot_dir`.  If the file is not found in any location, the script fails
with an informative message.

**REQ-04.8** — The configuration shall support a `progress_window` boolean
that controls whether `restore` shows a progress window by default.
This setting may be overridden per-run with `--show-window` or `--no-window`.

**REQ-04.9** — The configuration shall support a `user_scripts_folder` property
specifying the directory where the `restore` wrapper looks for user hook scripts.
The default shall be `~/.config/wayland-desktop/scripts`.  Relative paths shall
be resolved relative to the project root.  `~` shall be expanded.  The wrapper
shall fall back gracefully to the project root if the property cannot be read.
Two hook scripts are supported, both optional and both located in this folder:
`restore-user-before` (run after the startup dialogs, before `kwin_restore.py`
launches) and `restore-user-after` (run after `kwin_restore.py` returns,
regardless of success or failure).

---

### REQ-05 Layout snapshot

**REQ-05.1** — The system shall be capable of enumerating all currently open
user-facing windows and recording their placement (desktop, monitor, geometry,
state) to a YAML file.

**REQ-05.2** — System and compositor windows (plasmashell, ibus, krunner, etc.)
shall be excluded from snapshots by default.  The list of excluded window
classes shall be configurable.

**REQ-05.3** — Each snapshot entry shall include the command needed to relaunch
the application.  This command shall be derived automatically from the
application's `.desktop` file `Exec=` line where available.

**REQ-05.4** — Each snapshot entry shall be assigned a unique tracking UUID at
capture time.  This UUID shall persist in the YAML file and shall be re-used
on subsequent restores of the same file.

**REQ-05.5** — Snapshot files shall be human-readable and hand-editable.
Entries shall be grouped by virtual desktop in the order the desktops appear
in the configuration file.  Windows visible on all desktops shall appear last.

**REQ-05.6** — Snapshot files shall use aliases for desktop and monitor names
where aliases are defined, so that the file reads in user-friendly terms
rather than KWin connector names.

**REQ-05.7** — Windows that appear on all virtual desktops shall be represented
with `desktop: '*'` rather than a separate boolean flag.

**REQ-05.8** — Within each desktop group, snapshot entries shall be sorted
alphabetically by application alias.  This ensures that successive snapshots
of the same desktop produce consistently ordered YAML files, making diff-based
comparison between snapshots reliable and meaningful.

---

### REQ-06 Layout restore

**REQ-06.1** — The system shall be capable of reading a snapshot YAML file and
restoring all applications listed in it to their recorded desktop, monitor,
and state.

**REQ-06.2** — For single-instance applications, if a window is already open
at restore time, that window shall be placed without launching a new instance.

**REQ-06.3** — For multi-instance applications, a new instance shall always be
launched, even if one is already open on another desktop.

**REQ-06.4** — For tray-only applications (marked `ignore_window: true`), the
application shall be launched without waiting for a window to appear.

**REQ-06.5** — After placing each window, the system shall write an entry to the
session state file mapping the window's `tracking_uuid` to its `internalId`.

**REQ-06.6** — The restore operation shall support a dry-run mode that shows
what would be done without launching or moving any windows.

**REQ-06.7** — The restore operation shall support a filter option to restore
only specific applications from the snapshot.  The filter shall accept exact
app aliases, exact `resourceClass` values, and case-insensitive substrings as
a fallback, so that `chrome` matches `google-chrome`.

**REQ-06.8** — The restore operation shall support a `--no-launch` mode that
places only windows that are already open, without launching any new processes.

**REQ-06.9** — The restore script shall be suitable for invocation from a KDE
autostart entry on login, with `X-KDE-AutostartPhase=2` to ensure KWin is
fully initialised before the script runs.

**REQ-06.10** — The restore script shall not require a snapshot file to be
specified as a positional command-line argument.  Instead, the script shall
accept an optional `--setup FILE` argument.  If `--setup` is omitted, the
script shall read the `default_setup` property from `config.yaml` and locate
the named file by searching, in order: the current working directory,
`~/.config/wayland-desktop/`, and `snapshot_dir`.

**REQ-06.11** — The restore script shall accept an optional positional
argument naming a single application to restore (e.g.
`python3 restore chrome`).  This argument shall be treated as
equivalent to `--filter APP`.

**REQ-06.12** — The restore script shall optionally display a progress window
showing colour-coded log output in real time, a Verbose checkbox, and an
Abort/Close button.  The window shall run in a separate process so that the
Qt event loop does not interfere with the synchronous restore logic.

**REQ-06.13** — The restore script shall support a `--log FILE` option that
tees all stdout and stderr output to the specified file with ANSI colour codes
stripped.  Parent directories shall be created as needed.  The log file shall
begin with a header line containing the timestamp and the full command line
used to invoke the script.  The tee shall be installed before any other output
is produced so that the complete run is captured.

**REQ-06.14** — When the `restore` wrapper is invoked with a positional APP
argument, it shall suppress both user hook scripts (`restore-user-before` and
`restore-user-after`).  This allows users to test the placement of a single
application without triggering heavyweight setup tasks (such as mounting
network shares or starting VPN services) that the hook scripts typically
perform.

---

### REQ-07 Session tracking

**REQ-07.1** — Each managed window shall carry a `tracking_uuid` that
uniquely identifies it within the snapshot file it was restored from.

**REQ-07.2** — At restore time, after each window is successfully placed,
`restore` shall write an entry to a session state file
(`~/.config/wayland-desktop/session_state.json`) mapping the window's `tracking_uuid`
to its current `internalId`.  The session state file shall be updated
incrementally.

**REQ-07.3** — The session state file shall record, per entry: `tracking_uuid`,
`internalId`, `app_alias`, `resource_class`, `desktop`, `screen`, and a
`placed_at` ISO timestamp.

**REQ-07.4** — The system shall be capable of finding any managed window by
looking up its `tracking_uuid` in the session state file to obtain its
`internalId`, then scanning `workspace.windowList()` in a KWin script.

**Note:** Qt dynamic properties (`setProperty`/`property`) are not available
on window objects in the KWin JS scripting sandbox (confirmed 2026-03-21).
The session state file is the adopted alternative.  See ADR-260321-01.

---

### REQ-08 Window drift correction

**REQ-08.1** — A separate utility (`realign`) shall be provided that
re-places windows that have drifted from their intended desktop, monitor, or
position, without running a full restore.

**REQ-08.2** — The realign utility shall identify windows by looking up their
`tracking_uuid` in the session state file to obtain the current `internalId`.

**REQ-08.3** — The realign utility shall read the desired placement from the
snapshot YAML file used at restore time.

**REQ-08.4** — The realign utility shall be invocable on demand and optionally
on a timer.

**REQ-08.5** — The realign utility shall report windows whose `internalId` is
no longer present in `workspace.windowList()` as stale rather than as errors.

---

### REQ-09 Window tiling and keystroke control

**REQ-09.1** — The system shall support placing a window using KWin's built-in
tile positions (left half, right half, top half, bottom half, quadrants).

**REQ-09.2** — Tile operations shall use KWin's internal tiling slot methods
(e.g. `workspace.slotWindowQuickTileLeft()`) rather than fake keystroke
injection.

**REQ-09.3** — A `tile` field shall be supported in snapshot entries and in
`config.yaml` app definitions.  Accepted values: `left`, `right`, `top`,
`bottom`, `top-left`, `top-right`, `bottom-left`, `bottom-right`.

**REQ-09.4** — For application content setup (REQ-10), the system shall
support sending keyboard shortcuts and mouse clicks to a specific window by
`internalId`.  The preferred implementation is a C++ KWin Effect Plugin
exposing a D-Bus interface.  See PROBLEMS.md for a full evaluation of all
available options.

**Status:** REQ-09.1–09.3 (tile/snap) are **complete** — `tile()` in
`window.py` and the `tile:` field in snapshot YAML and `config.yaml` are
all implemented and tested.  REQ-09.4 (keystroke/mouse injection) is
**complete** — implemented via the XDG RemoteDesktop portal in
`kwin_lib/input_injection.py` and `restore` phase 4.  See
ADR-260323-01 and `docs/dev/INVESTIGATION_LOG.md`.

---

### REQ-10 Application content setup

**REQ-10.1** — A separate setup script framework shall be provided that,
after window placement is complete, can configure the internal state of
applications — loading URLs in browser windows, setting up terminal tab
layouts in Konsole, and running initial commands in terminal tabs.

**REQ-10.2** — Konsole tab setup shall support: creating named tabs, running
a specified command in each tab, and setting the working directory per tab.
This shall use Konsole's D-Bus interface (`org.kde.konsole`) or its
`--profile` and `--workdir` command-line arguments.

**REQ-10.3** — Browser setup shall support opening a specified set of URLs in
a browser window's tabs.

**REQ-10.4** — The setup framework shall be separable from the window
placement system: placement runs first, then content setup runs against
the already-placed windows.

**REQ-10.5** — Content setup specifications shall be stored in the snapshot
YAML file or in a separate companion file.

**Status:** REQ-10.1 and REQ-10.3 are partially implemented via the
`input_actions` DSL field in snapshot entries.  The DSL supports typing
URLs into browser address bars (`{CTRL+L}url{ENTER}`), sending keyboard
shortcuts, and mouse clicks.  REQ-10.2 (Konsole tab setup via D-Bus) and
REQ-10.4–10.5 (framework separability) are planned but not yet designed.

---

## Non-functional requirements

**NFR-01 Platform** — The system shall run on OpenSUSE Tumbleweed with KDE
Plasma 6 on Wayland.  It shall not depend on X11 tools or protocols.

**NFR-02 Language** — All library and script code shall be written in Python 3.13.

**NFR-03 Dependencies** — Runtime dependencies shall be limited to packages
available in the standard OpenSUSE Tumbleweed repositories:
`python313-pydbus`, `python313-gobject`, `python313-PyYAML`.
The progress window and click indicator additionally require `python313-qt6`
(PyQt6).  Mouse click capture (`mouse-clicks`) additionally requires
`python-evdev` (available via pip) and membership in the `input` group.

**NFR-04 Reliability** — Restore operations shall be tolerant of individual
application failures.  If one application fails to launch or cannot be placed,
the restore shall continue with the remaining entries and report failures at
the end.

**NFR-05 Idempotency** — Running the restore script multiple times against
the same snapshot shall produce the same result as running it once.

**NFR-06 Human-readable files** — All configuration and snapshot files shall
be YAML, readable and editable without knowledge of the library internals.

**NFR-07 Code conventions** — All non-trivial code shall include explanatory
comments.  All generated files shall include a signed header identifying the
generating tool (e.g. `# Generated by: Claude Sonnet 4.6`).

**NFR-08 Logging** — All CLI scripts shall produce clear, colour-coded progress
output to the terminal.  A progress window provides the same output in a GUI
during restore.

---

## Constraints

**CON-01** — Window resize is not possible via the Wayland compositor.
Maximization and tile operations are the reliable alternatives.

**CON-02** — Screen assignment uses `workspace.sendClientToScreen(window, output)`.
`w.output` is read-only and `sendToOutput()` does not exist.  `sendClientToScreen()`
silently ignores minimized windows — the window must be unminimized first.

**CON-03** — `w.output.name` does not update within the same KWin script
execution after `sendClientToScreen()` is called (stale property).  Screen
moves are fire-and-trust.  Similarly, tile slot result geometry is not reliably
readable within the same script execution.

**CON-04** — Virtual desktop assignment requires live QObject references from
`workspace.desktops`; integer numbers or UUID strings alone are not accepted.

**CON-05** — Only one `KWinSession` instance may be active at a time per
user session, as each instance holds the `org.kwinlib.Session` D-Bus name.
`WindowProbe` uses a separate service name and is not subject to this constraint.

**CON-06** — Setting `onAllDesktops = true` on a Wayland window triggers
KWin geometry propagation to all desktops.  This must be applied last in
any sequence of geometry operations or the per-desktop stored geometry will
be incorrect.

**CON-07** — Cross-process input injection (sending keystrokes or mouse clicks
to a specific window) requires the XDG RemoteDesktop portal
(`org.freedesktop.portal.RemoteDesktop`).  KWin's private EIS interface
(`org.kde.KWin.EIS.RemoteDesktop`) withholds `KEYBOARD` capability.  The
portal requires a one-time user authorization dialog; a restore token is
persisted to skip the dialog on subsequent runs.

---

## Out of scope

- **Application internal state restore** (open files, editor state, browser
  history) — this is a separate problem requiring per-application support.

- **Flatpak application support** — Flatpak-sandboxed applications may expose
  different `resourceClass` values and PID visibility to KWin.  Basic support
  may work but is not explicitly tested or guaranteed.

- **Multi-user / system-wide configuration** — the library is designed for a
  single user's desktop session only.

- **X11 session compatibility** — the library is Wayland-only.

- **KDE Activities** — KDE Plasma supports Activities as a second orthogonal
  dimension of workspace organisation, independent of virtual desktops.  The
  current library does not capture or restore Activity assignment.  It is not
  yet known whether the KWin scripting API exposes Activity membership on
  window objects, or whether Activity state interacts with the `onAllDesktops`
  geometry drift issue.  Activities support is deferred pending investigation.

- **Automatic native Wayland session restore** — KDE Plasma is developing
  session restore functionality via the `xx-session-management-v1` protocol,
  tracked by KDE bugs #15329 and #436318.  However, this wayland-desktop project
  already provides more functionality than what is anticipated in the planned
  Wayland session restore — input action automation and Secret Service integration.
  This software is therefore likely to remain useful for a while, even after native session restore
  is implemented.

---

## Implementation status

| Requirement | Status | Notes |
|---|---|---|
| REQ-01.1 Window identification | ✓ Complete | `probe.py`, `probe` |
| REQ-01.2 Existing window detection | ✓ Complete | `relaunch=False` path in `probe.py` |
| REQ-01.3 Tracking UUID | ✓ Complete | UUID generated at snapshot time; session state file written at restore time |
| REQ-01.4 Timing tolerance | ✓ Complete | `windowAdded` signal, configurable timeout |
| REQ-01.5 Splash/dialog filtering | ✓ Complete | Pre-launch snapshot diff |
| REQ-01.6 Tray app support | ✓ Complete | `ignore_window` in config, snapshot, and restore |
| REQ-02.1 Move to desktop | ✓ Complete | `WindowManager.move_to_desktop()` |
| REQ-02.2 Move to monitor | ✓ Complete | `WindowManager.move_to_screen()` via `sendClientToScreen()` |
| REQ-02.3 All desktops sentinel | ✓ Complete | `desktop: '*'` / `"all"` |
| REQ-02.4 Maximize on correct screen | ✓ Fixed | Three-phase restore with settle delay; ADR-260321-02 |
| REQ-02.5 Absolute position | ✓ Complete | `WindowManager.move()` |
| REQ-03.1–03.4 State control | ✓ Complete | maximize, minimize, unmaximize, unminimize, restore, raise |
| REQ-03.5 Tile/snap | ✓ Complete | `WindowManager.tile()`; `tile:` field in snapshot and config |
| REQ-04 Configuration and aliases | ✓ Complete | `config.py`, `config.yaml` |
| REQ-04.7 Default setup file | ✓ Complete | `default_setup` in config |
| REQ-04.8 Progress window config | ✓ Complete | `progress_window` in config; `--show-window` / `--no-window` |
| REQ-04.9 User scripts folder | ✓ Complete | `user_scripts_folder` in config; `restore-user-before` and `restore-user-after` hooks; read by `restore` via `getconfig` |
| REQ-05 Layout snapshot | ✓ Complete | `snapshot.py`, `snapshot` |
| REQ-06.1–06.4 Restore core | ✓ Complete | Five-phase restore in `restore` |
| REQ-06.5 Session state written on restore | ✓ Complete | `session_state.py` |
| REQ-06.6 Dry-run | ✓ Complete | `--dry-run` |
| REQ-06.7 Filter with substring fallback | ✓ Complete | `--filter` / positional arg; case-insensitive substring match |
| REQ-06.8 No-launch mode | ✓ Complete | `--no-launch` |
| REQ-06.9 Autostart | ✓ Documented | README.md |
| REQ-06.10–06.11 Setup file / single-app arg | ✓ Complete | `--setup`, `resolve_setup_file()`, positional `APP` |
| REQ-06.12 Progress window | ✓ Complete | `kwin_lib/progress_window.py`; `--show-window` / `--no-window` |
| REQ-06.13 Log file | ✓ Complete | `--log FILE`; `_TeeWriter` / `_install_log_tee()` in `restore` |
| REQ-06.14 Single-app suppresses hooks | ✓ Complete | `restore` bash wrapper; positional arg detection loop |
| REQ-07 Session tracking | ✓ Complete | `session_state.py`; ADR-260321-01 |
| REQ-08 Drift correction | ✓ Complete | `realign`; screen map keyed by `id(entry)` to handle duplicate aliases |
| REQ-09.1–09.3 Tile/snap positioning | ✓ Complete | `WindowManager.tile()`; ADR-260322-04 |
| REQ-09.4 Keystroke/mouse injection | ✓ Complete | XDG RemoteDesktop portal; `input_actions` DSL; phase 4 in `restore`; ADR-260323-01 |
| REQ-10 Application content setup | ~ Partial | `input_actions` DSL covers browser URL loading and keyboard shortcuts |
