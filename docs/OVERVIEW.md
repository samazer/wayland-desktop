<!-- SPDX-FileCopyrightText: 2026 Sam Azer <sam dot azer at azertech dot net>  -->
<!-- SPDX-License-Identifier: CC-BY-SA-4.0  -->
# wayland-desktop — How It Works

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

- [Copyright &amp; License](#copyright--license)
- [The fundamental problem](#the-fundamental-problem)
- [Architecture](#architecture)
- [The five-phase restore](#the-five-phase-restore)
- [Drift correction](#drift-correction)
- [Window identification](#window-identification)
- [Screen assignment](#screen-assignment)
- [Why window resize does not work](#why-window-resize-does-not-work)
- [Input injection (phase 4)](#input-injection-phase-4)
- [Session tracking](#session-tracking)
- [Progress window](#progress-window)

---

### Copyright &amp; License
Wayland Desktop Restore (documentation) Copyright © 2026 by Sam Azer &lt;sam dot azer at azertech dot net&gt; is licensed under Creative Commons Attribution-ShareAlike 4.0 International. To view a copy of this license, visit https://creativecommons.org/licenses/by-sa/4.0/

#### Creative Commons Attribution-ShareAlike 4.0 International
This license requires that re-users give credit to the creator. It allows re-users to distribute, remix, adapt, and build upon the material in any medium or format, even for commercial purposes. If others remix, adapt, or build upon the material, they must license the modified material under identical terms.

- Credit must be given to the creator.
- Adaptations must be shared under the same terms.

[See the full License](https://creativecommons.org/licenses/by-sa/4.0/)

---

## The fundamental problem

Development of the Session Restore functionality in Wayland is delayed.
Manually restoring a complicated desktop session can be a time-consuming
process, leading to repeated thoughts of automation.

KDE Plasma 6 on Wayland removed the X11-era tools (`wmctrl`, `xdotool`,
`xprop`) that were previously used for scripted window management.  The
Wayland protocol deliberately restricts what external processes can do to
windows — a client application must consent to being moved or resized.

KWin's own **JavaScript scripting engine** is not subject to these
restrictions, because it runs inside the compositor process itself.  This
library exploits that by injecting short-lived JavaScript scripts into the
running KWin instance and receiving results back via D-Bus callbacks.

## Architecture

```
kwin_lib Python script
    │
    ├─ Publishes a D-Bus receiver service on the session bus
    │   (org.kwinlib.Session)
    │
    ├─ Writes a temporary .js file to /tmp
    │
    ├─ Calls /Scripting loadScript() to load it into KWin
    │
    └─ KWin executes the script inside its QJSEngine:
           workspace.windowList()            — enumerate windows
           workspace.windowAdded             — signal for new windows
           workspace.desktops                — virtual desktops
           workspace.currentDesktop = d      — switch active desktop
           workspace.screens                 — connected monitors
           w.desktops = [desktopObj]         — move to desktop
           workspace.sendClientToScreen(w,s) — move to screen
           w.setMaximize(h, v)               — maximize
           w.minimized = true/false          — minimize/unminimize
           workspace.activeWindow = w        — raise/focus
           workspace[slotName]()             — tile operations
           callDBus(...)                     — send results back
```

The library-side Python script runs a GLib main loop in a background thread
to dispatch incoming D-Bus method calls from KWin scripts.  All communication
is asynchronous but the API presents a synchronous interface using
`threading.Event` for synchronisation.

`ScriptRunner.run()` measures elapsed time from script load to callback and
injects `elapsed_ms` into every result dict.  Every JS `send()` call includes
at minimum `{ ok: bool, op: string }` plus op-specific fields.

## The five-phase restore

`restore` runs in five phases separated by settle delays:

**Phase 1 — Launch and identify:**
Each app is launched (or an existing window found). Immediately after
identification, `unmaximize()` then `minimize()` is called on the window.
The unmaximize clears geometry the app may have self-applied during its own
session restore (e.g. Brave restoring to its own full-screen size on the
wrong screen). The minimize parks the window out of the way for the
duration of phases 2 and 3, preventing visual noise and compositor
interference during moves.

**Phase 2 — Move:**
Each window is unminimized, moved to its target desktop and screen, then
re-minimized. The unminimize/re-minimize cycle is required because
`workspace.sendClientToScreen()` silently ignores minimized windows.

**Settle delay (default 3 seconds):**
Allows KWin to commit all screen re-associations before phase 3.

**Phase 3 — Finalise:**
Each window applies its target state using one of four paths, in priority order:

1. **minimize** — the window is already minimized from phase 2; call
   `set_on_all_desktops()` if needed and return immediately.  No KWin
   window operations are issued.
2. **maximize** — call `maximize()` directly (which implicitly unminimizes),
   then `raise_window()`.  A 250ms sleep before `maximize()` gives KWin
   time to be ready after the settle delay expires.  The geometry check
   is skipped because `maximize()` always fills the screen.
3. **tile** — `unminimize()`, shrink check (safety net for oversized
   windows), `tile()`, then `raise_window()`.
4. **raise** (default) — `unminimize()`, shrink check, `raise_window()`.

`raise_window()` is called after every non-minimize operation so the
window is guaranteed to be on top.  `set_on_all_desktops()` is called
last — see ADR-260321-03.

**Phase 4 — Input actions:**
For each placed window that has a non-empty `input_actions` field in its
snapshot entry, the DSL sequence is executed: keystrokes, mouse events,
sleeps.  Phase 4 is skipped entirely if no entries have `input_actions`.
Individual action failures are non-fatal and logged as warnings.

The XDG RemoteDesktop portal session is opened **before Phase 1** (not at
the start of Phase 4) so that any stale-token KDE authorization dialog appears
at a known, calm moment rather than mid-restore.  Similarly, the Secret Service
connection is primed before Phase 1 so that KeePassXC approval dialogs surface
early.  The `restore` bash wrapper shows two additional `kdialog` prompts (Ready
/ Watch for pop-ups) before Python is even invoked.

Before phase 4 begins, `kwin_restore.py` switches back to the desktop that was
active when the script started, putting the user's desktop in a known state
before any input actions fire.

The progress window is hidden for the entire duration of Phase 4 so it cannot
obscure tool windows, KeePassXC dialogs, or the windows being acted upon.

See the Input injection section below and `docs/SNAPSHOT.md` for the DSL
reference.

**Phase 5 — Re-tile all-desktops windows:**
For each window placed with `on_all_desktops=True`, loop over every
virtual desktop in order. On each desktop: `switch_to_desktop()`,
`move_to_screen()`, then tile/maximize/raise as per the snapshot entry.
This corrects the KWin-side behaviour where tile position is stored per
*(window, desktop)* pair — a window placed on desktop 1 and then set to
`on_all_desktops` will be correctly tiled only on desktop 1 without this
pass.  After the loop, the originally active desktop is restored.
Phase 5 is skipped silently if no all-desktops windows were placed.

**After all phases:** `kwin_restore.py` unconditionally calls
`logger.show_window()` (guaranteeing the progress window is visible even if
phase 4 left it hidden) and then switches back to the desktop that was active
when the script started.  This final switch is in addition to the pre-phase-4
switch; the two serve different purposes — the earlier one establishes a known
state *before* input actions fire, the later one returns the user to their
starting desktop *after* everything has completed.  On error, both operations
are also attempted in the `except` handler before the error is reported.

After `kwin_restore.py` returns, the `restore` bash wrapper runs the optional
`restore-user-after` hook script (if present) before exiting.

## Drift correction

After the restore, windows may drift over time — particularly after the
screen saver unlocks, after extended inactivity, or when KWin recomposes.
`realign` corrects this without relaunching any applications.

**Loop structure:**

```
outer loop: each virtual desktop in order
    switch_to_desktop(desktop)
    inner loop: each entry in the YAML setup file
        default mode:  process only on_all_desktops entries
        --all-apps:    also process single-desktop entries on their own desktop
        for each entry: move_to_screen → tile/maximize/minimize/raise
restore the originally active desktop
```

**Window lookup:** `realign` does not launch anything.  It finds
windows by:
1. Looking up `tracking_uuid → internalId` in
   `~/.config/wayland-desktop/session_state.json` (written by `restore`
   after phase 1).
2. Falling back to a live `workspace.windowList()` scan by `resourceClass`
   if the session state entry is absent or stale.

The reason the outer loop is over desktops (not entries) is that KWin
stores tile position per *(window, desktop)* pair.  The script must be on
the target desktop at the moment it issues the tile command for the tile
to land in the correct slot.

## Window identification

Windows are identified by their `internalId` — a UUID that KWin assigns when
a window is created.  The watcher script hooks the `workspace.windowAdded`
signal (fired inside KWin the moment a new window is created) and reports the
UUID back to `probe.py` via `callDBus()`.

A snapshot of existing windows is taken before launch.  Any window that was
already present when the script loaded is excluded, ensuring only the newly
created window is returned.

## Screen assignment

`workspace.sendClientToScreen(window, output)` is the correct API.  The
output object is found by iterating `workspace.screens` by connector name.

**Important:** `sendClientToScreen()` silently ignores minimized windows.
`__w.output.name` does not update within the same script execution after the
call (stale property).  Both behaviours are fire-and-trust — see ADR-260322-04.

## Why window resize does not work

On Wayland, window dimensions are negotiated between the application and the
compositor — the compositor cannot unilaterally impose a size on a client.
`frameGeometry` position (`x`, `y`) can be set and takes effect immediately.
`frameGeometry` size (`width`, `height`) is silently ignored by the application.

The practical workarounds are `setMaximize()` (uses the standard Wayland XDG
activation protocol that applications are required to honour) and the tile
slot methods (`slotWindowQuickTileLeft` etc.) which snap the window to a
half or quarter of the screen via KWin's own tiling engine.

## Input injection (phase 4)

Sending keystrokes and mouse events to a specific window requires the
**XDG RemoteDesktop portal** (`org.freedesktop.portal.RemoteDesktop`).

KWin's private EIS interface (`org.kde.KWin.EIS.RemoteDesktop`) was
investigated first but found to withhold `KEYBOARD` capability — its seat
only advertises `TOUCH`.  The portal is the correct path.

**Session lifecycle:**

1. `CreateSession` — async Request/Response handshake, returns a session path
2. `SelectDevices` — requests `KEYBOARD | POINTER` with `persist_mode=2`
3. `Start` — shows the KDE "Allow wayland-desktop to control input?" dialog
   (first run only); returns a `restore_token`
4. `NotifyKeyboard*` / `NotifyPointer*` — synchronous D-Bus calls per event
5. `Close` — releases the session on exit

The restore token is saved to `~/.config/wayland-desktop/portal_session_token`.
On subsequent runs it is passed as `restore_token` to `SelectDevices`,
which causes `Start` to skip the dialog entirely.

**Important implementation notes** (see ADR-260323-01 for full detail):

- All three setup calls are async — they return a request path immediately
  and fire a `Response` signal when complete.  A GLib main loop is required.
- Gio `signal_subscribe` callbacks take **7** arguments (the last is `user_data`).
- Pre-constructing `GLib.Variant("a{sv}", {})` and passing it inside another
  Variant produces type `v` (wrapped), not `a{sv}` (inline).  Always pass `{}`
  directly as a Python dict and let PyGObject infer from the outer format string.

**Portal methods used:**

| Method | Purpose |
|---|---|
| `NotifyKeyboardKeycode(session, {}, keycode, state)` | Key press / release |
| `NotifyPointerMotion(session, {}, dx, dy)` | Pointer move (relative) |
| `NotifyPointerButton(session, {}, button, state)` | Mouse button |
| `NotifyPointerAxisDiscrete(session, {}, axis, steps)` | Scroll |

`NotifyPointerMotionAbsolute` was evaluated but requires a PipeWire
screen-capture stream node ID that is not available outside a screen-capture
session.  Instead, `NotifyPointerMotion` (relative) is used.  A virtual cursor
position is maintained in `InputInjector` and seeded from KWin's reported
cursor location before each injection sequence; relative deltas are computed
from that tracked position, making absolute targeting reliable without the
PipeWire dependency.

## Session tracking

Each snapshot entry carries a `tracking_uuid` generated at capture time.
At restore time, `restore` writes `~/.config/wayland-desktop/session_state.json`
mapping each `tracking_uuid` to the live `internalId` of the placed window.
This file enables `realign` to re-place drifted windows by UUID lookup
without re-running a full restore.

Qt dynamic properties (`setProperty`/`property`) are not available on window
objects in the KWin JS sandbox — confirmed by testing on 2026-03-21.

## Progress window

`restore` optionally shows a PyQt6 progress window that displays
colour-coded log output in real time, with a Verbose checkbox and an
Abort/Close button.  The window runs in a spawned child process; the parent
communicates via `multiprocessing.Queue` and `multiprocessing.Value` objects.
The window saves and restores its geometry via
`~/.config/wayland-desktop/progress_window.json`.

Enable via `--show-window`, suppress via `--no-window`, or set
`progress_window: true` in `config.yaml`.
