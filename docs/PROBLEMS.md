<!-- SPDX-FileCopyrightText: 2026 Sam Azer <sam dot azer at azertech dot net>  -->
<!-- SPDX-License-Identifier: CC-BY-SA-4.0  -->
# wayland-desktop — Known Limitations and Open Issues

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
- [Known Limitations](#known-limitations)
- [Open Issues](#open-issues)

---

### Copyright &amp; License
Wayland Desktop Restore (documentation) Copyright © 2026 by Sam Azer &lt;sam dot azer at azertech dot net&gt; is licensed under Creative Commons Attribution-ShareAlike 4.0 International. To view a copy of this license, visit https://creativecommons.org/licenses/by-sa/4.0/

#### Creative Commons Attribution-ShareAlike 4.0 International
This license requires that re-users give credit to the creator. It allows re-users to distribute, remix, adapt, and build upon the material in any medium or format, even for commercial purposes. If others remix, adapt, or build upon the material, they must license the modified material under identical terms.

- Credit must be given to the creator.
- Adaptations must be shared under the same terms.

[See the full License](https://creativecommons.org/licenses/by-sa/4.0/)

---

## Known Limitations

These are fundamental constraints imposed by the Wayland protocol or the
current KWin API.  They are not bugs — they reflect the boundaries of what
the compositor permits external scripts to do.

| Limitation | Root cause | Workaround |
|---|---|---|
| Window **resize** not possible | Wayland: compositor cannot force app to resize | Use `maximize()` or `tile()` instead |
| `w.output` is read-only | KWin JS sandbox does not expose `sendToOutput()` | Use `workspace.sendClientToScreen(w, output)` — confirmed working |
| `__w.output.name` stale after move | KWin does not update the property within the same script execution | Fire-and-trust: no read-back after `sendClientToScreen()` or tile slot calls |
| `setProperty`/`property` unavailable | `QObject` methods not surfaced by KWin JS wrapper | Session state file maps `tracking_uuid` → `internalId` instead |
| Cross-process input injection restricted | Wayland protocol: input events must go through the focused window | XDG RemoteDesktop portal (`org.freedesktop.portal.RemoteDesktop`) — one-time user auth dialog; restore token persisted for subsequent runs. Fully implemented in `kwin_lib/input_injection.py`. See ADR-260323-01. |
| Only one `KWinSession` active at a time | `org.kwinlib.Session` D-Bus name is exclusive per user session | `WindowProbe` uses a separate name; not subject to this constraint |

---

## Open Issues

Confirmed bugs and unimplemented features, ranked by impact.

---

### ~~1. Realign utility — not yet implemented (REQ-08)~~  RESOLVED

`realign` is written and working.  It corrects window drift without
relaunching any applications by iterating over virtual desktops and re-placing
each tracked window from the session state file.  The screen map is keyed by
`id(entry)` to correctly handle setups where multiple YAML entries share the
same screen alias.  See `realign` and `docs/OVERVIEW.md` (Drift
correction section) for full details.

---

### ~~2. Persistent logging — not yet implemented (REQ-NFR-08 extension)~~  RESOLVED

`restore` now accepts a `--log FILE` option.  All output (stdout and
stderr) is teed to the specified file with ANSI colour codes stripped.  Parent
directories are created as needed.  The log file opens with a header line
containing the timestamp and full command line.  The tee is installed as the
very first step in `main()` so that the header and all subsequent output,
including startup messages, are captured.

For autostart use, pass `--log` to the `restore` wrapper:

```ini
Exec=/path/to/wayland-desktop/restore --log ~/.config/wayland-desktop/restore.log
```

---

### ~~3. Keystroke and mouse injection — not yet implemented (REQ-09.4)~~  PARTIALLY RESOLVED

Implemented via the XDG RemoteDesktop portal.  See ADR-260323-01 and
`kwin_lib/input_injection.py`.  The `input_actions` DSL field in snapshot
entries drives phase 4 of `restore`.  See `docs/SNAPSHOT.md` for
the full DSL reference and `docs/dev/INVESTIGATION_LOG.md` for the
diagnostic investigation that determined this was the correct approach.

**Resolved:** Keyboard injection (`{ENTER}`, `{CTRL+T}`, `{PASS key}`, plain
text typing, and all key token families) is working correctly.

**Remaining open issue:** Mouse click injection (`{WCLICK}`, `{CLICK}`,
`{RCLICK}`, `{WDRAG}`, etc.) is not working properly.  The portal session
opens successfully and the `NotifyPointerMotion` / `NotifyPointerButton` calls
are issued without error, but clicks do not reliably land at the intended
position or register with the target application.  The root cause has not yet
been determined.  As a workaround, replace mouse-based `input_actions` sequences
with equivalent keyboard shortcuts where possible.

The `--show-clicks` visual indicator overlay is also affected and should not be
relied upon until click injection is resolved.

---

### ~~4. input_actions passwords are stored in plain text~~  RESOLVED

The `{PASS key}` DSL token resolves passwords at restore time from the Secret
Service (so far tested only on KeePassXC) rather than storing them in the snapshot YAML file.
Passwords are held in memory only for the duration of the `send_char()` loop
that types them — they are never written to disk or logged.  Enable with
`use_secret_service: true` in `config.yaml`.  See `docs/CONFIG.md` and
`kwin_lib/secret_service.py` for full details.

---

### 5. KDE Activities — not investigated (out of scope)

KDE Plasma supports Activities as a second orthogonal workspace dimension.
The current library does not capture or restore Activity assignment.
Investigation questions are documented in REQUIREMENTS.md (Out of scope).

---

### 6. Phase 3 unmaximize — redundant and may break some apps

Phase 3 (`finalise_entry`) previously called `unmaximize()` immediately
after `unminimize()`, before applying the target state.  This was
originally added to clear oversized geometry that apps such as Brave
self-apply during their own session restore.

However, Phase 1 already calls `unmaximize()` followed by `minimize()`
immediately after each window is identified, giving the app the full
duration of Phase 2 and the settle delay to process the compositor's
configure event.  By the time Phase 3 runs the window should already
be in a normal unmaximized state, making the Phase 3 `unmaximize()`
redundant.

The redundant call is suspected to cause layout artifacts or state
confusion in apps that do not aggressively self-restore their geometry.

**Current state:** The Phase 3 `unmaximize()` call has been removed.
Testing required against Brave (the original motivating case) and a
representative set of other apps.  If Brave regresses, the fix is to
introduce a per-app `geometry_algo` config flag (`default` | `brave`)
rather than restoring the blanket Phase 3 unmaximize.

---

### 7. Menu navigation via arrow keys is unreliable in input_actions

When navigating application menus using arrow keys in `input_actions` DSL
sequences, menu items are sometimes skipped.  This appears to be a timing
issue: the injected arrow-key events arrive before the menu has fully
rendered all its items, causing the focus to land on the wrong entry.

**Workaround:** Assign a keyboard shortcut directly to the desired menu
action (via the application's own shortcut configuration) and invoke it
by shortcut rather than navigating with arrow keys.  This is more robust
and less fragile to application or system load variations.

---

### 8. Secret Service approval dialogs appear on unpredictable screens

KeePassXC's approval pop-ups (the "Allow access to passwords?" dialog)
can appear on any virtual desktop and any screen, and frequently appear
beneath other windows.  When a dialog is hidden the restore appears to
hang — `restore` is waiting for the user to approve, but the
dialog is not visible.

`restore` mitigates this by priming the Secret Service connection
before Phase 1 (fetching and discarding each required password once to
trigger all approval dialogs at a known, calm moment).  However, the
dialogs can still appear in inconvenient locations.

**Workaround:** If the restore appears to hang during the priming step
(logged as `Secret Service: primed ...` lines before Phase 1), check all
virtual desktops and look for a KeePassXC dialog hidden behind other
windows.  In KeePassXC's settings, enabling "Remember approval for this
application" reduces how often the dialog appears.

---

### 9. Verify that wayland/ scripts can be run from any working directory

The bash wrapper scripts (`restore`, `snapshot`, `realign`, etc.) locate the
`wayland/` directory relative to their own path via `BASH_SOURCE[0]`, so they
work correctly from any working directory.

However, the Python scripts inside `wayland/` (`kwin_restore.py`,
`kwin_snapshot.py`, etc.) each call
`sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))` to locate
`kwin_lib/`.  This should allow them to be invoked directly from any directory
using an absolute or relative path (e.g.
`python3 /path/to/wayland-desktop/wayland/restore`), but this has not
been formally verified across all scripts and invocation styles.

**Action needed:** Test each `wayland/kwin_*.py` script when invoked directly
with a full path from a non-project working directory, and confirm `kwin_lib`
imports succeed.  If any script fails, adjust its `sys.path` setup accordingly.

---

## Input Injection — Investigation and Resolution (REQ-09.4)

**Status: RESOLVED.** Implemented 2026-03-23.  See ADR-260323-01.

### What was investigated

Six options were evaluated (A through F below), plus the KWin private EIS
interface which was discovered during implementation.

### Option G — XDG RemoteDesktop portal ✓ ADOPTED

`org.freedesktop.portal.RemoteDesktop` (xdg-desktop-portal-kde6) provides
`NotifyKeyboardKeycode`, `NotifyPointerMotionAbsolute`, `NotifyPointerButton`,
and `NotifyPointerAxisDiscrete` as direct D-Bus method calls.

| Property | Detail |
|---|---|
| Mechanism | D-Bus calls to `org.freedesktop.portal.Desktop` |
| Privileges | None — one-time user authorization dialog |
| Targeting | Focus-based: raise target window first, then inject |
| Wayland compliance | Fully compliant — uses the standard portal API |
| Persistence | `persist_mode=2` issues a restore token; dialog never repeats |
| Token file | `~/.config/wayland-desktop/portal_session_token` |
| libei / ctypes | Not required |

This is the **implemented solution**.  See OVERVIEW.md for the session
lifecycle and ADR-260323-01 for all implementation findings including
the GLib Variant pitfalls.

### KWin private EIS interface — investigated but insufficient

`org.kde.KWin.EIS.RemoteDesktop` (confirmed path `/org/kde/KWin/EIS/RemoteDesktop`)
was investigated.  It returns a libei file descriptor via `connectToEIS(caps)`
but the seat only advertises `TOUCH` capability — `KEYBOARD` is
deliberately withheld.  This was confirmed by `ei_seat_has_capability` probe:

```
KBD-ONLY EVENT type=3  seat=yes (caps=['TOUCH'])  device_caps=NULL
```

No amount of `ei_seat_bind_capabilities(KEYBOARD)` calls produces a keyboard
device.  The private interface is not viable for keyboard injection.

See `docs/dev/INVESTIGATION_LOG.md` for complete diagnostic output.

### Option A — ydotool

`ydotool` injects events via `/dev/uinput` through a daemon (`ydotoold`).
Requires `input` group membership.  Events go to whichever window has focus
at delivery time — targeting is focus-based only.  Not viable for background
injection; superseded by Option G.

### Option B — C++ KWin Effect Plugin

A KWin Effect Plugin running inside the compositor process can inject input
directly without portal authorization.  Requires `kwin6-devel` build chain.
Not needed given Option G works without compilation.  Remains the
recommended path if targeted background-window injection (non-focused) is
ever required.

### Option C — xdotool

X11 only — not viable for native Wayland clients on KDE Plasma 6.

### Option D — KWin D-Bus input simulation

No input injection surface exposed on `org.kde.KWin` — not viable.

### Option E — wtype / wlrctl

`zwlr_virtual_keyboard_v1` is a wlroots protocol — not implemented by KWin.
Not viable on KDE Plasma.

### Option F — Konsole D-Bus interface

Konsole exposes `org.kde.konsole` D-Bus interface for direct session/tab
control without any input injection.  Correct approach for Konsole tab setup
specifically.  Not a general solution.

### Summary

| Option | Targeting | Viable | Status |
|---|---|---|---|
| G — XDG RemoteDesktop portal | Focus-based | ✓ | **Implemented** |
| KWin private EIS | Focus-based | ✗ — no keyboard | Investigated, insufficient |
| A — ydotool | Focus-based | Partial | Superseded by G |
| B — KWin Effect Plugin | Precise by internalId | ✓ | Available if needed |
| C — xdotool | X11 only | ✗ | Not viable |
| D — KWin D-Bus | Not available | ✗ | Not viable |
| E — wtype/wlrctl | wlroots only | ✗ | Not viable |
| F — Konsole D-Bus | Konsole only | ✓ | For Konsole specifically |
