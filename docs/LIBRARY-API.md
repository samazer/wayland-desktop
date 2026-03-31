<!-- SPDX-FileCopyrightText: 2026 Sam Azer <sam dot azer at azertech dot net>  -->
<!-- SPDX-License-Identifier: CC-BY-SA-4.0  -->
# wayland-desktop — Library API Reference

Python API for the `kwin_lib` package.  All classes and functions are
importable from `kwin_lib` directly.

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
- [KWinSession](#kwinsession)
- [WindowProbe](#windowprobe)
- [WindowManager](#windowmanager)
- [DesktopManager](#desktopmanager)
- [InputInjector](#inputinjector)
- [SecretService](#secretservice)
- [ClickIndicator](#clickindicator)
- [ProgressWindow and ProgressWindowLogger](#progresswindow-and-progresswindowlogger)
- [launch\_and\_place](#launch_and_place)

---

### Copyright &amp; License
Wayland Desktop Restore (documentation) Copyright © 2026 by Sam Azer &lt;sam dot azer at azertech dot net&gt; is licensed under Creative Commons Attribution-ShareAlike 4.0 International. To view a copy of this license, visit https://creativecommons.org/licenses/by-sa/4.0/

#### Creative Commons Attribution-ShareAlike 4.0 International
This license requires that re-users give credit to the creator. It allows re-users to distribute, remix, adapt, and build upon the material in any medium or format, even for commercial purposes. If others remix, adapt, or build upon the material, they must license the modified material under identical terms.

- Credit must be given to the creator.
- Adaptations must be shared under the same terms.

[See the full License](https://creativecommons.org/licenses/by-sa/4.0/)

---

## KWinSession

The main entry point.  Use as a context manager; it owns all shared
infrastructure and cleans up on exit.

```python
from kwin_lib import KWinSession

with KWinSession() as kwin:
    # kwin.desktops  — DesktopManager
    # kwin.windows   — WindowManager
    # kwin.probe     — WindowProbe
    pass
```

Raises `RuntimeError` if KWin is not reachable or if another `KWinSession` is
already open in the same process.

---

## WindowProbe

Launches a program and returns the `WindowInfo` of its new window.  Handles
the case where the app is already running.

```python
from kwin_lib import WindowProbe

probe = WindowProbe(
    timeout_secs = 30.0,     # how long to wait for the window to appear
    filter_class = "",       # only accept windows with this resourceClass
    stop_count   = 1,        # stop after this many windows are detected
    relaunch     = False,    # if True, launch even if app already running
)

pid, windows = probe.run(["thunderbird"])
# pid == 0 means app was already running; existing window returned
# windows is a list of WindowInfo objects

uuid = windows[0].internalId
```

When `relaunch=False` (the default), `WindowProbe` checks whether a window
matching `filter_class` (or the inferred class from the command name) is
already open.  If so, it returns that window without launching a new process.
This is the correct behaviour for most boot-time restore scenarios.

---

## WindowManager

Operates on windows identified by their `internalId` UUID.

```python
with KWinSession() as kwin:
    wm = kwin.windows

    # Find all windows for an app
    wins = wm.find_by_class("thunderbird-esr")

    # Get current info for a known UUID
    info = wm.get_info("{d6a03aac-...}")

    # Move to a virtual desktop
    desktop = kwin.desktops.find_desktop(name="Main")
    wm.move_to_desktop(uuid, desktop)

    # Move to a specific screen
    screen = kwin.desktops.find_screen(name="HDMI-A-2")
    wm.move_to_screen(uuid, screen)

    # State changes
    wm.maximize(uuid)
    wm.unmaximize(uuid)
    wm.minimize(uuid)
    wm.restore(uuid)          # un-minimize and un-maximize
    wm.raise_window(uuid)     # bring to front and focus

    # Compound operation: desktop + screen + state in one call
    wm.place(
        uuid,
        desktop   = kwin.desktops.find_desktop(name="Main"),
        screen    = kwin.desktops.find_screen(name="HDMI-A-2"),
        maximize  = True,
        raise_win = True,
    )

    # Make a window visible on ALL virtual desktops
    wm.place(uuid, desktop="*")                    # using sentinel string
    wm.place(uuid, desktop="all")                  # equivalent
    wm.place(uuid, on_all_desktops=True)           # equivalent keyword form

    # All-desktops + specific screen + maximize
    wm.place(uuid,
             desktop          = "*",
             screen           = kwin.desktops.find_screen(name="HDMI-A-2"),
             maximize         = True)

    # Tile snap
    wm.tile(uuid, "left")         # half-screen left
    wm.tile(uuid, "top-right")    # quarter top-right

    # Geometry verification after maximize.
    # Reads back window and screen geometry; retries unmaximize+maximize if
    # the window did not visually fill the screen within tolerance.
    # Note: geometry read-back after setMaximize() is currently unreliable
    # in the KWin JS sandbox — see PROBLEMS.md issue 6.
    wm.ensure_maximized(
        uuid,
        width_tolerance  = 0,    # from cfg.maximize_width_tolerance
        height_tolerance = 50,   # from cfg.maximize_height_tolerance
        logger           = None, # optional ProgressWindowLogger for warnings
    )
```

All methods raise `RuntimeError` if the window is not found or an operation
fails.

---

## DesktopManager

Enumerates and looks up virtual desktops and screens.

```python
with KWinSession() as kwin:
    dm = kwin.desktops

    # List all virtual desktops
    desktops = dm.list_desktops()
    for d in desktops:
        print(d.name, d.number, d.id)

    # Find by name, number, or UUID
    d = dm.find_desktop(name="Main")
    d = dm.find_desktop(number=1)
    d = dm.find_desktop(id="4ca398c8-...")

    # Current desktop UUID
    current_id = dm.current_desktop_id()

    # Switch the active (visible) desktop
    dm.switch_to_desktop(d, settle_s=0.4)
    # settle_s: seconds to wait after the switch before issuing further
    # KWin commands (tile/move operations require the switch to be committed)

    # Query the active window and cursor position in one round-trip
    info = dm.active_window_and_cursor()
    # Returns: { active_id, caption, cursor_x, cursor_y }
    # Used by mouse-clicks to identify the clicked window

    # List all screens
    screens = dm.list_screens()
    for s in screens:
        print(s.name, s.x, s.y, s.width, s.height)

    # Find screen by connector name or index
    s = dm.find_screen(name="HDMI-A-2")
    s = dm.find_screen(index=0)

    # Which screen contains a given point?
    s = dm.screen_for_point(2000, 1200)
```

---

## InputInjector

Opens an XDG RemoteDesktop portal session and injects keyboard and pointer
events.  Used by `restore` phase 4 to execute `input_actions` DSL
sequences.

On first use a KDE "Allow wayland-desktop to control your keyboard and pointer?"
authorization dialog is shown; the user approves once.  A restore token is
saved to `~/.config/wayland-desktop/portal_session_token` and reused on subsequent
runs so the dialog never appears again.  If the token is stale the dialog
reappears and the old token is deleted automatically.

```python
from kwin_lib.input_injection import InputInjector, run_input_actions, parse_actions

# Preferred: open early (before Phase 1) so any auth dialog fires at startup
inj = InputInjector()
inj.open()
# ... run phases 1-3 ...
run_input_actions(result, inj, kwin, logger)
# ... more entries ...
inj.close()

# Alternative: context-manager form (opens on entry, closes on exit)
with InputInjector() as inj:
    run_input_actions(result, inj, kwin, logger)
```

**Methods:**

| Method | Description |
|---|---|
| `open() -> InputInjector` | Open the portal session. Raises `RuntimeError` if the portal is unreachable or the user denies authorization. |
| `close() -> None` | Close the portal session and release all resources. |
| `init_cursor(kwin)` | Seed the virtual cursor position from KWin's reported cursor location. Call before the first pointer move in each injection sequence. |
| `send_key(keycode, modifiers)` | Press modifier keys, press/release key, release modifiers. |
| `send_char(char)` | Type a single character (looks up keycode and shift state from the built-in US QWERTY map). |
| `send_pointer_abs(x, y)` | Move the pointer to absolute compositor coordinates using relative motion deltas computed from the tracked virtual cursor. |
| `send_button(button, pressed)` | Press or release a mouse button (`BTN_LEFT=0x110`, `BTN_RIGHT=0x111`, `BTN_MIDDLE=0x112`). |
| `send_scroll(direction)` | Scroll `"up"` or `"down"` one notch. |
| `focus_window(uuid, kwin)` | Raise and focus the window identified by `uuid`, then sleep 300 ms. |
| `inject_actions(actions, uuid, kwin, logger, ...)` | Execute a list of `Action` objects against the given window. |

**DSL helpers:**

| Function | Description |
|---|---|
| `parse_actions(dsl: str) -> list[Action]` | Parse an `input_actions` DSL string into a list of `Action` dataclasses. Raises `ValueError` on unknown tokens. |
| `run_input_actions(result, injector, kwin, logger, ...)` | Phase 4 driver: focus the window, verify focus, then call `inject_actions` for each parsed action. |

---

## SecretService

Retrieves passwords from the Secret Service (so far tested only on KeePassXC) at restore time for
`{PASS key}` DSL tokens.  Requires `use_secret_service: true` in `config.yaml`
and KeePassXC running with Secret Service Integration enabled.

```python
from kwin_lib.secret_service import SecretService

ss = SecretService()
ss.open()                             # opens D-Bus connection to the Secret Service
password = ss.get_password("my-key") # looks up entry by title (label)
ss.close()
```

**Methods:**

| Method | Description |
|---|---|
| `open() -> None` | Open a D-Bus connection to `org.freedesktop.Secret.Service`. Raises `RuntimeError` if the service is unreachable. |
| `get_password(key: str) -> str` | Return the secret for the entry whose label exactly matches `key`. Unlocks the item if necessary. Raises `RuntimeError` if the key is not found or the item cannot be unlocked. |
| `close() -> None` | Release the D-Bus connection. |

**Notes:** Passwords are held in memory only for the duration of the
`send_char()` loop that types them.  They are never written to disk or logged.
Call `open()` before Phase 1 (not at Phase 4) so any KeePassXC approval dialogs
appear at startup — see OVERVIEW.md.

---

## ClickIndicator

> ⚠️ **Not currently working.** The `ClickIndicator` overlay is implemented but
> does not function correctly in practice.  Mouse click injection via the XDG
> RemoteDesktop portal is itself not yet working (see PROBLEMS.md issue 3), so
> the visual indicator has not been usable or verifiable.  The `--show-clicks`
> option to `restore` is present but should not be relied upon until both the
> click injection and the overlay are resolved.

Full-screen transparent overlay that draws a brief red dot with a white
crosshair at each injected pointer position during phase 4.  Activated only
when `--show-clicks` is passed to `restore`.

```python
from kwin_lib.click_indicator import ClickIndicator

ci = ClickIndicator()
ci.start()               # spawn the overlay process
ci.show_click(x, y)      # draw a 48px dot at compositor position (x, y)
ci.hide()                # hide the overlay (e.g. while a password dialog is open)
ci.show()                # restore the overlay
ci.stop()                # destroy the overlay process
```

**Methods:**

| Method | Description |
|---|---|
| `start() -> None` | Spawn the overlay child process and wait for it to be ready. |
| `show_click(x, y)` | Display a 48 px red dot at absolute compositor coordinates `(x, y)` for 1 second. |
| `hide() -> None` | Hide the overlay window without stopping the child process. Used during `{PASS}` token execution so the overlay cannot obscure KeePassXC dialogs. |
| `show() -> None` | Restore the overlay window after `hide()`. |
| `stop() -> None` | Terminate the overlay child process. |

The overlay sets `keepAbove`, `onAllDesktops`, and `skipTaskbar` via KWin JS
so it is always visible above other windows on all virtual desktops and does
not appear in the taskbar or alt-tab switcher.

---

## ProgressWindow and ProgressWindowLogger

`ProgressWindow` is a PyQt6 window that displays colour-coded log output in
real time.  It runs in a spawned child process; the parent communicates via
`multiprocessing.Queue` and `multiprocessing.Value`.  `ProgressWindowLogger`
is a thin wrapper used by the restore phases — it accepts the same `info`,
`ok`, `warn`, `error`, and `head` calls regardless of whether a window is
present.

```python
from kwin_lib.progress_window import ProgressWindow, ProgressWindowLogger

win = ProgressWindow()
win.show(initial_verbose=False)
logger = ProgressWindowLogger(win)

logger.head("Phase 1")
logger.info("Launching thunderbird")
logger.ok("Window identified")
logger.warn("Desktop not found")
logger.error("Fatal error")

# Hide the window during phase 4 so it cannot obscure tool windows
logger.hide_window()
# ... run injection ...
logger.show_window()

# Signal completion and wait for the user to close the window
win.set_complete()
win.wait_for_close()
```

**`ProgressWindow` methods:**

| Method | Description |
|---|---|
| `show(initial_verbose)` | Spawn the child process and display the window. |
| `set_complete()` | Enable the Close button and change Abort to Close. |
| `wait_for_close()` | Block until the user closes the window. |
| `is_showing() -> bool` | True if the window child process is alive. |
| `is_verbose() -> bool` | Current state of the Verbose checkbox. |
| `is_abort_requested() -> bool` | True if the user clicked Abort. |

**`ProgressWindowLogger` methods:**

| Method | Description |
|---|---|
| `info(msg)` | Log an `[INFO ]` line (cyan). |
| `ok(msg)` | Log an `[OK   ]` line (green). |
| `warn(msg)` | Log a `[WARN ]` line (yellow). Also prints to stderr. |
| `error(msg)` | Log an `[ERROR]` line (red). Also prints to stderr. |
| `head(msg)` | Log a bold cyan section heading. |
| `hide_window()` | Send `_CMD_HIDE` to the child — the window disappears from screen and taskbar. No-op if no window. |
| `show_window()` | Send `_CMD_SHOW` to the child — the window reappears. No-op if no window. |

The window saves and restores its geometry via
`~/.config/wayland-desktop/progress_window.json`.

---

## launch\_and\_place

Module-level convenience function for the common case of launching an app and
placing it on a specific desktop and screen in one call.

```python
from kwin_lib import launch_and_place

win = launch_and_place(
    "thunderbird",
    desktop      = "Main",       # alias or canonical name
    screen       = "main",       # alias or connector name
    maximize     = True,         # default True
    timeout_secs = 30.0,
    relaunch     = False,        # reuse existing window if open
)

# Make the window visible on all virtual desktops
win = launch_and_place("brave", desktop="*",   screen="main")
win = launch_and_place("brave", desktop="all", screen="main")  # equivalent
```

Uses `KWinSession` internally; do not call this inside an existing
`KWinSession` block.
