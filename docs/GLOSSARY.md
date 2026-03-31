<!-- SPDX-FileCopyrightText: 2026 Sam Azer <sam dot azer at azertech dot net>  -->
<!-- SPDX-License-Identifier: CC-BY-SA-4.0  -->
# wayland-desktop — Glossary

Terms used in the codebase, documentation, and snapshot files.  Where a term
has both a colloquial meaning and a specific technical meaning within this
project, the technical meaning is given.

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


## Contents

- [Copyright &amp; License](#cl)
- [A](#a) — alias, all-desktops sentinel, app
- [C](#c) — canonical name, config.yaml, connector name
- [D](#d) — desktop, desktop alias, drag, dry-run
- [F](#f) — filter, fire-and-trust
- [G](#g) — geometry, getconfig
- [I](#i) — internalId, input_actions, ignore_window
- [K](#k) — KWin script, KWinSession
- [M](#m) — minimize, multi-instance
- [O](#o) — onAllDesktops
- [P](#p) — phase, portal session token, progress window
- [R](#r) — resourceClass, restore, restore token, restore-user-before, restore-user-after
- [S](#s) — screen alias, secret-service, settle delay, single-instance, snapshot, session state
- [T](#t) — tile, tracking UUID
- [U](#u) — user_scripts_folder
- [V](#v) — virtual desktop
- [W](#w) — window

---

### Copyright &amp; License {cl}
Wayland Desktop Restore (documentation) Copyright © 2026 by Sam Azer &lt;sam dot azer at azertech dot net&gt; is licensed under Creative Commons Attribution-ShareAlike 4.0 International. To view a copy of this license, visit https://creativecommons.org/licenses/by-sa/4.0/

#### Creative Commons Attribution-ShareAlike 4.0 International
This license requires that re-users give credit to the creator. It allows re-users to distribute, remix, adapt, and build upon the material in any medium or format, even for commercial purposes. If others remix, adapt, or build upon the material, they must license the modified material under identical terms.

- Credit must be given to the creator.
- Adaptations must be shared under the same terms.

[See the full License](https://creativecommons.org/licenses/by-sa/4.0/)

---

## A

**alias**
A short, user-defined name that maps to a canonical KWin name.  Aliases are
defined in `config.yaml` for desktops, screens, and applications.  They are
bidirectional: once `Job1` is defined as an alias for `"Job 1"`, either form is
accepted everywhere.  Unknown names pass through unchanged without error.
See also: *canonical name*.

**all-desktops sentinel**
The special value `"*"` (or equivalently `"all"`, case-insensitive) that, when
used as a desktop name, causes a window to appear on every virtual desktop
simultaneously rather than being assigned to one specific desktop.  Represented
in snapshot files as `desktop: '*'`.  No alias definition is required — these
values are built-in.
See also: *virtual desktop*, *on_all_desktops*.

**app** *(snapshot field)*
The application alias from `config.yaml`, or the `resourceClass` if no alias is
defined.  Used as the human-readable label for a snapshot entry.  When passing
an app name to `--filter` or as a positional argument to `restore`,
both the alias and the `resourceClass` are accepted.
See also: *alias*, *resourceClass*.

**AppConfig**
The Python dataclass (`kwin_lib/config.py`) that holds per-application
configuration: `resource_class`, `single_instance`, and an optional
`command_override`.  One `AppConfig` instance exists per entry in the `apps:`
section of `config.yaml`.

---

## C

**canonical name**
The exact string that KWin uses internally to identify a desktop, screen, or
application — as opposed to a user-defined alias.  Examples: `"Job 1"` (desktop
canonical name), `"HDMI-A-2"` (screen canonical name), `"thunderbird-esr"`
(application `resourceClass`).  The library always works with canonical names
internally; aliases are resolved at the CLI/user-facing boundary.
See also: *alias*.

**callDBus()**
The KWin JavaScript function used to send data from a running KWin script back
to the Python caller.  The primary communication mechanism from KWin to the
library.  Used to return window lists, window-added events, and operation
results.  See ADR-260317-03.

**compositor**
The software responsible for drawing the desktop and managing windows at the
display level.  On KDE Plasma 6 / Wayland, KWin is the compositor.  Because
the compositor owns all windows, it is the only process permitted to move,
resize, or change the desktop assignment of windows belonging to other
applications.  This is why the library injects JavaScript into KWin rather than
manipulating windows from outside.
See also: *KWin*, *Wayland*.

**connector name**
The identifier used by the operating system and GPU driver for a physical
display output port.  Examples: `"HDMI-A-2"`, `"DP-4"`, `"Unknown-2"`.
Connector names are reported by KWin as `w.output.name` and appear in snapshot
files as the `screen` canonical value.  They can change after a reboot if the
GPU driver re-enumerates outputs in a different order.
See also: *screen*, *alias*.

---

## D

**D-Bus**
The inter-process communication system used throughout the Linux desktop
session.  KWin exposes its scripting interface on the session D-Bus bus under
the service name `org.kde.kwin.Scripting`.  The library publishes its own
receiver services (`org.kwinlib.Session`, `org.probe.KWinProbe`) on the same
bus to receive callbacks from injected scripts.

**desktop**
Short for *virtual desktop*.  A named workspace that contains a set of windows.
The user switches between desktops to see different sets of windows.  In KDE
Plasma, desktops are sometimes called "Activities" in casual usage, but
Activities are a distinct feature — see *Activities*.
See also: *virtual desktop*, *desktop UUID*, *on_all_desktops*.

**desktop UUID**
The internal UUID string that KWin assigns to each virtual desktop.  Used
internally by the library to match desktop objects in JavaScript scripts.
Distinct from the *tracking UUID* assigned by this library to windows.
See also: *virtual desktop*, *tracking UUID*.

**desktopFileName** *(KWin property)*
The base name of the `.desktop` file associated with an application window, as
reported by KWin (`w.desktopFileName`).  Examples: `"thunderbird-esr"`,
`"org.kde.konsole"`.  Used by the snapshot module to locate the application's
`.desktop` file and extract its `Exec=` line for the `command` field.
See also: *command*.

**drift**
The phenomenon where a window moves away from its intended desktop, monitor, or
geometry after an extended idle period or screen unlock event.  The planned
`realign` utility addresses drift by re-placing windows identified by
their *tracking UUID* without running a full restore.
See also: *tracking UUID*, *realign*.

**drag** *(input_actions DSL)*
A pointer press-move-release sequence captured by `mouse-clicks` and
replayed by `kwin_lib/input_injection.py`.  Dragging is detected when a
left-button press and release are separated by more than `--drag-threshold`
pixels in either axis (default: 5 px).  Three drag tokens are supported,
following the same `W`-prefix convention as click tokens:

| Token | Button | Coordinates |
|---|---|---|
| `{WDRAG x1,y1 to x2,y2}` | Left | Window-relative — emitted by `mouse-clicks` |
| `{WRDRAG x1,y1 to x2,y2}` | Right | Window-relative |
| `{DRAG x1,y1 to x2,y2}` | Left | Absolute compositor coordinates |

During replay, the injector moves the pointer to the start position, presses
the button, moves to the end position, then releases.  50 ms pauses are
inserted between each phase to allow the application to register the event.
See also: *input_actions*, `{WCLICK}`.

---

## F

**filter**
A restriction applied to a restore operation that limits which snapshot entries
are processed.  Specified via `--filter APP` (repeatable) or the positional
`APP` argument.  Both the app alias and `resourceClass` are accepted as filter
values.

**frameGeometry** *(KWin property)*
The rectangle `{x, y, width, height}` that defines a window's position and size
in the compositor coordinate space.  Setting `w.frameGeometry` in a KWin script
moves the window.  On Wayland, the compositor can request a new position but
cannot force a resize; the application may ignore size changes.  The `x` and
`y` fields in a snapshot entry come from `frameGeometry`.

---

## G

**geometry**
The position and size of a window, expressed as `x, y, width, height` in
the compositor coordinate space.  The coordinate space spans all connected
monitors; the origin (0, 0) is typically the top-left corner of the leftmost
monitor.  Geometry is always stored in snapshot files even for maximised windows,
because it represents the window's *restored* (un-maximised) size and position.
See also: *frameGeometry*, *compositor coordinate space*.

**getconfig**
The bash wrapper script (and the underlying `wayland/getconfig`) that
provides read-only inspection of `~/.config/wayland-desktop/config.yaml` from
the command line.  Supports two commands: `list` (prints all top-level keys and
values) and `lookup <key>` (navigates a dot-separated path such as
`apps.brave.single_instance` and prints the value with no trailing newline,
suitable for shell `$()` capture).  Used by `restore` to read `user_scripts_folder`
before each run.  Falls back gracefully when no config file exists.

---

## I

**ignore_window**
A boolean field in `config.yaml` (per-app) and in snapshot YAML entries.
When `true`, the application launches directly to the system tray and never
creates a visible main window.  `restore` launches such apps with
`subprocess.Popen` and immediately moves on without waiting for a window to
appear and without performing any placement.  The canonical example is
KeePassXC.  The `ignore_window` flag is read from `config.yaml` at snapshot
capture time and written into the YAML entry so that restore behaviour is
preserved even when the config is not available.
See also: *tray app*, *single-instance*.

**input_actions** *(snapshot field)*
A string field in a snapshot YAML entry containing a sequence of actions to
execute against the window after it has been placed, during phase 4 of
`restore`.  Written by hand; `snapshot` always emits an empty
string.  Parsed and executed by `kwin_lib/input_injection.py` via the XDG
RemoteDesktop portal.

The field uses a simple DSL that mixes plain text (typed character by character)
with special tokens enclosed in `{…}`.  Supported token families:

- **Key tokens** — `{ENTER}`, `{ESC}`, `{TAB}`, `{F1}`…`{F12}`, `{CTRL+T}`, etc.
- **Mouse click tokens** — `{WCLICK x,y}`, `{WRCLICK x,y}`, `{CLICK x,y}`,
  `{RCLICK x,y}`, `{MCLICK x,y}`
- **Mouse drag tokens** — `{WDRAG x1,y1 to x2,y2}`, `{WRDRAG x1,y1 to x2,y2}`,
  `{DRAG x1,y1 to x2,y2}`
- **Scroll tokens** — `{SCROLLUP}`, `{SCROLLDOWN}`
- **Control tokens** — `{SLEEP 500ms}`, `{FOCUS}` / `{FOCUS-MAIN}` (raise main
  window), `{ACTIVE-WINDOW}` (capture currently focused dialog as tool window),
  `{FOCUS-TOOL}` (raise captured tool window), `{PASS key}` (fetch secret from
  Secret Service and type it)

`W`-prefixed tokens use window-relative coordinates (offset from the window's
top-left corner at replay time).  Unprefixed tokens use absolute compositor
coordinates.  Comment lines beginning with `#` are silently ignored, so output
from `mouse-clicks` can be pasted directly without editing.

See `docs/SNAPSHOT.md` for the full DSL reference and examples.
See also: *phase*, *portal session token*, *{PASS}*.

**internalId**
A UUID string assigned by KWin to each window for the duration of its
existence in the current session.  The primary handle used by the library to
target a specific window in JavaScript operations.  Changes each time the window
is opened; does not survive a reboot.  Distinct from the *tracking UUID* that
the library stamps onto windows at restore time.
See also: *tracking UUID*.

---

## K

**KWin**
The window manager and compositor for KDE Plasma.  On Wayland, KWin is the
sole authority over window placement and state.  All window operations in this
library are implemented as JavaScript snippets loaded into KWin via its
`/Scripting loadScript()` D-Bus interface.
See also: *compositor*, *KWin scripting*.

**KWin Effect Plugin**
A C++ shared library (`.so`) loaded into the KWin process at startup via the
KWin plugin architecture.  Effect plugins run inside the compositor and have
access to `KWin::EffectWindow` objects, which expose `setData(role, value)` /
`data(role)` for arbitrary per-window storage.  This is the C++ equivalent of
the `setProperty`/`property` mechanism that was found to be unavailable in the
KWin JS sandbox.  Evaluated as an alternative window-tagging mechanism in
ADR-260321-01; deferred in favour of the session state file for the current
implementation, but remains the preferred path if in-process tagging is needed
in the future.  Build requires: `kwin6-devel`, `cmake`, `gcc-c++`,
`extra-cmake-modules`.
See also: *session state file*, *KWin scripting*.
The JavaScript execution environment embedded in KWin (`QJSEngine`).  Scripts
loaded via `/Scripting loadScript()` run inside the compositor process with
full access to the `workspace` object and all window QObjects.  This is the
only mechanism available for cross-process window management on Wayland.
See also: *workspace*, *callDBus()*.

**KWinSession**
The primary Python context manager (`kwin_lib/__init__.py`) that wraps a live
connection to KWin.  Holds a `ScriptRunner`, a `DesktopManager`, and a
`WindowManager`.  Only one `KWinSession` may be active at a time per user
session, as each holds the `org.kwinlib.Session` D-Bus name.

---

## M

**maximized**
A window state in which the window fills its monitor.  On Wayland, maximization
is the most reliable mechanism for making a window fill a screen, because
direct resize via `frameGeometry` is not enforceable.  A window can be
horizontally maximized, vertically maximized, or both.  In snapshot files,
`maximized: true` means fully maximized (both axes).

**monitor**
A physical display device.  In this project's documentation, "monitor" and
*screen* are used interchangeably.  The KWin API uses "output" for the same
concept.
See also: *screen*, *connector name*.

**multi-instance**
An application configured with `single_instance: false` (the default).  At
restore time, a new instance is always launched, even if one is already open.
Appropriate for terminal emulators and file managers that may need to run on
multiple desktops simultaneously.
See also: *single-instance*.

---

## O

**on_all_desktops**
A boolean property of a KWin window (`w.onAllDesktops`) that, when true, makes
the window visible on every virtual desktop.  In snapshot files, this state is
represented as `desktop: '*'` rather than a separate `on_all_desktops` field.
Setting `onAllDesktops` on a Wayland window can cause KWin to re-negotiate
window geometry; see PROBLEMS.md issue 3.
See also: *all-desktops sentinel*.

---

## P

**placement**
The act of moving a window to a specific virtual desktop and screen, and
applying its state (maximised, minimised, etc.).  In the library, `place()` in
`window.py` is the single method that combines all these operations in the
correct sequence: unmaximise → move to desktop → move to screen → apply state.

**portal session token**
A short string returned by the XDG RemoteDesktop portal's `Start` call after
the user approves the "Allow wayland-desktop to control your keyboard and
pointer?" dialog.  Saved to `~/.config/wayland-desktop/portal_session_token` and
passed as `restore_token` to `SelectDevices` on subsequent runs, causing the
authorization dialog to be skipped entirely.  If the token becomes stale (the
portal rejects it with `response=1`), `InputInjector` deletes it automatically
and retries, triggering the dialog once more to obtain a fresh token.
See also: *InputInjector*, *input_actions*.

**probe** *(verb / noun)*
To launch an application and wait for its window to appear, returning a
`WindowInfo` handle.  Also refers to the `WindowProbe` class and `probe`
script (invoked via the `probe` wrapper) that implement this operation.
See also: *WindowProbe*.

---

## R

**realign**
The operation performed by `realign` that re-places windows that have
drifted from their intended desktop, monitor, or position, without running a
full restore and without relaunching any applications.  Identifies windows by
looking up their *tracking UUID* in the session state file to obtain the current
`internalId`, then issuing move/tile/maximize/raise operations on each desktop
in turn.  The outer loop iterates over virtual desktops (not entries) because
KWin stores tile position per *(window, desktop)* pair — the compositor must be
on the target desktop at the moment the tile command is issued.
See also: *drift*, *tracking UUID*, *session state file*.

**resourceClass** *(KWin property)*
The string KWin uses to identify the application that owns a window, derived
from the application's WM_CLASS X11 property or Wayland equivalent.  Examples:
`"thunderbird-esr"`, `"org.kde.konsole"`, `"brave-browser"`.  The primary
identifier used to match windows to config entries and to filter restore
operations.  Different from the window title and from the `desktopFileName`.

**resourceName** *(KWin property)*
The instance part of the WM_CLASS pair, as opposed to `resourceClass` (the
class part).  Less useful for identification; `resourceClass` is preferred for
all matching operations.

**restore**
The operation of reading a snapshot YAML file and re-placing all listed
applications on their recorded desktops, screens, and states.  Implemented by
`restore`, invoked via the `restore` bash wrapper.

**restore-user-before**
An optional executable script placed in the `user_scripts_folder` directory.
When present, the `restore` wrapper runs it after the startup dialogs succeed
but before `kwin_restore.py` launches.  Typical use: start background services
(VPNs, SSH agents, KeePassXC) that the restore depends on.  The `getconfig`
and `secret-service` wrappers are available inside `restore-user-before`.
See also: *restore-user-after*, *user_scripts_folder*, *getconfig*, *secret-service*.

**restore-user-after**
An optional executable script placed in the `user_scripts_folder` directory.
When present, the `restore` wrapper runs it after `kwin_restore.py` returns,
regardless of whether `kwin_restore.py` succeeded or failed.  The exit code of
`kwin_restore.py` is captured before the hook runs and re-applied afterwards, so
the hook cannot mask a restore failure.  Typical use: post-restore tasks such as
opening a terminal, sending a desktop notification, or cleaning up temporary
state created by `restore-user-before`.
See also: *restore-user-before*, *user_scripts_folder*.

---

## S

**screen**
A physical display, identified in the KWin API by its *connector name*
(`w.output.name`).  In `config.yaml`, screens are mapped from a friendly alias
to a connector name.  In snapshot files, the `screen` field uses the alias if
one is defined, or the connector name directly.
See also: *connector name*, *alias*, *monitor*.

**secret-service**
The bash wrapper script (and the underlying `wayland/secret-service`)
that provides `list` and `lookup` access to the Secret Service
(`org.freedesktop.secrets`, implemented by KeePassXC) from the command line
and from `restore-user-before` scripts.  `lookup <key>` prints the secret for
the entry whose label matches `key`, with no trailing newline, so that
`SECRET=$(secret-service lookup "my-key")` captures cleanly.
See also: *{PASS}*, *restore-user-before*, *SecretService*.

**ScriptRunner**
The Python class (`kwin_lib/bus.py`) responsible for writing a JavaScript
snippet to a temporary file, loading it into KWin via D-Bus, waiting for the
callback, and unloading the script.  The core of the library's KWin interaction
mechanism.

**session** *(Plasma session)*
The running KDE Plasma desktop session for the current user.  Distinct from a
*KWinSession* (the library's context manager object).  The Plasma session begins
at login and ends at logout or reboot.  Window `internalId` values and *tracking
UUIDs* are valid only within a single Plasma session.

**session state file** (`session_state.json`)
A JSON file written by `restore` to `~/.config/wayland-desktop/session_state.json`
after each successful window placement.  Maps each `tracking_uuid` to the live
`internalId` of the placed window, along with metadata (`app_alias`,
`resource_class`, `desktop`, `screen`, `placed_at` timestamp).  Read by
`realign` to locate managed windows by UUID without requiring in-process
window tagging.  Entries are session-scoped: `placed_at` timestamps allow the
realign utility to detect and prune entries from previous sessions.  Updated
incrementally on each restore run — entries for windows not included in the
current run are preserved.
See also: *tracking UUID*, *internalId*, *realign*.

**setup file**
A snapshot YAML file used as the source for a restore operation.  The term
"setup" is used in the CLI (`--setup`) to emphasise that the file describes a
desired desktop configuration, not just a historical record.
See also: *snapshot*, *default_setup*.

**single-instance**
An application configured with `single_instance: true` in `config.yaml`.  At
restore time, if a window with the matching `resourceClass` is already open, it
is placed without launching a new copy.  Appropriate for applications that
should only ever have one instance running (e.g. Thunderbird, Obsidian).
See also: *multi-instance*.

**snapshot**
A YAML file recording the placement of all open windows at a moment in time:
their desktop, screen, geometry, state, and the command needed to relaunch them.
Produced by `snapshot`.  Used as input to `restore`.
See also: *setup file*, *LayoutSnapshot*.

**LayoutSnapshot**
The Python dataclass (`kwin_lib/snapshot.py`) representing a complete window
layout snapshot in memory.  Contains a list of `SnapshotEntry` objects, plus
metadata (timestamp, hostname).  Serialises to and from YAML via `save()` and
`load()`.

**SnapshotEntry**
The Python dataclass (`kwin_lib/snapshot.py`) representing one window in a
layout snapshot.  Contains all fields needed to identify, launch, and place the
window: `resource_class`, `app_alias`, `desktop`, `screen`, geometry, state
flags, `command`, `single_instance`, and `tracking_uuid`.

---

## T

**tray app**
An application that launches directly to the system tray notification area
and never displays a main window during startup.  KeePassXC is the primary
example.  Because `WindowProbe` waits for a window to appear, tray apps must
be handled separately: they are launched with `subprocess.Popen` and the
restore script immediately moves on without waiting or performing any placement.
Identified in `config.yaml` by `ignore_window: true`.
See also: *ignore_window*.

**tracking UUID** (`tracking_uuid`)
A UUID4 string assigned to each snapshot entry at capture time and stored in
the `tracking_uuid` field of the YAML file.  At restore time, after a window is
successfully placed, `restore` writes a mapping of this UUID to the
window's current `internalId` into the *session state file*.  The realign
utility looks up the `internalId` from that file to find the window in
`workspace.windowList()` without needing to know its title, class, or current
geometry.

Note: an earlier design attempted to stamp this UUID onto the KWin window
object as a Qt dynamic property (`setProperty("kwinlib_uuid", uuid)`).  Testing
on 2026-03-21 confirmed that `setProperty` is not available in the KWin JS
sandbox.  The session state file is the adopted alternative.  See ADR-260321-01.

Distinct from the *desktop UUID* assigned by KWin to virtual desktops.
See also: *desktop UUID*, *internalId*, *session state file*, *realign*.

---

## U

**user_scripts_folder**
A `config.yaml` top-level property that specifies the directory where the
`restore` wrapper looks for the `restore-user-before` and `restore-user-after`
hook scripts.  Defaults to `~/.config/wayland-desktop/scripts`.  Relative paths
are resolved against the project root (the directory containing `restore`).
`~` is expanded.  Read at runtime via `getconfig lookup user_scripts_folder`;
if getconfig fails, the project root is used as a fallback.
See also: *restore-user-before*, *restore-user-after*, *getconfig*.

**{PASS key}** *(input_actions DSL token)*
A DSL control token that looks up the Secret Service entry whose label exactly
matches `key` and types the returned secret character by character into the
focused window.  No Enter key is appended — add `{ENTER}` explicitly if needed.
Requires `use_secret_service: true` in `config.yaml` and KeePassXC running with
Secret Service Integration enabled.  The password is fetched at restore time,
held in memory only for the duration of the typing loop, and never written to
disk or logged.  The Secret Service connection is primed before Phase 1 so that
any KeePassXC approval dialogs appear at startup rather than mid-restore.

```yaml
input_actions: |
  {PASS router-admin}{ENTER}
  {PASS sam@example.com}
```

See also: *secret-service*, *input_actions*, *SecretService*.

---

## V

**virtual desktop**
A named workspace that groups a set of windows.  KDE Plasma supports multiple
virtual desktops; the user switches between them to see different sets of
windows.  Internally, KWin identifies each desktop by a *desktop UUID* and an
integer number; the library always works with desktop names (canonical or alias)
and resolves them to live QObject references before assignment.
See also: *desktop*, *desktop UUID*, *all-desktops sentinel*.

---

## W

**Wayland**
The display server protocol used by KDE Plasma 6.  Unlike X11, Wayland
explicitly forbids external processes from moving, resizing, or otherwise
manipulating windows belonging to other applications.  Only the compositor
(KWin) has that authority.  This constraint is the fundamental reason the
library injects JavaScript into KWin rather than using `wmctrl` or `xdotool`.
See also: *compositor*, *KWin*.

**window**
An application-owned surface managed by the compositor.  Each window has an
`internalId`, a `resourceClass`, a `desktopFileName`, a geometry, a set of
virtual desktops it appears on, and an output (screen) it is associated with.
In this library, windows are referred to by their `internalId` in all
compositor-facing operations.
See also: *internalId*, *resourceClass*, *WindowInfo*.

**WindowInfo**
The Python dataclass (`kwin_lib/models.py`) returned by `WindowProbe` after
successfully identifying a window.  Contains `internalId`, `resourceClass`,
`caption`, `pid`, desktop name, screen name, and geometry.

**WindowProbe**
The Python class (`kwin_lib/probe.py`) that launches an application, connects
to KWin's `workspace.windowAdded` signal, and returns a `WindowInfo` for the
newly created window.  Uses a pre-launch snapshot diff to distinguish new
windows from pre-existing ones, making it robust against both fast and slow
application startup.
See also: *probe*, *WindowInfo*.

**workspace** *(KWin JS object)*
The global object available in KWin JavaScript scripts that provides access to
the compositor's window list, virtual desktop list, screen list, and signals.
The entry point for all KWin scripting operations: `workspace.windowList()`,
`workspace.desktops`, `workspace.windowAdded`, `workspace.activeWindow`, etc.

---

*Terms are listed alphabetically within each letter section.  Cross-references
use italics.  New terms should be added here when they are introduced in the
codebase or documentation.*
