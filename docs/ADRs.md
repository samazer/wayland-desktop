<!-- SPDX-FileCopyrightText: 2026 Sam Azer <sam dot azer at azertech dot net>  -->
<!-- SPDX-License-Identifier: CC-BY-SA-4.0  -->
# wayland-desktop — Architectural Decision Records

**Project:** KDE Plasma 6 / Wayland Window Management Library  
**Format:** ADR-YYMMDDNN (two-digit year, ISO date, sequential number per date)  

**Policy:** ADRs are immutable records.  Once written, an ADR is never edited
or removed.  If a later decision supersedes an earlier one, a new ADR is added
that references the old one.  The old ADR may have a "Superseded by" note
appended, but its original text is preserved unchanged.  This ensures the full
decision history and the reasoning at each point in time is always recoverable.

New decisions should be appended with the next sequential number for that date,
e.g. ADR-260401-01 for the first decision taken on 2026-04-01.

---

## Available Documents
| Document                           | Description                                  |
|:-----------------------------------|:---------------------------------------------------|
| [README.md](../README.md)          | Installation, configuration, CLI tools             |
| [OVERVIEW.md](OVERVIEW.md)         | How the library works                              |
| [INSTALL.md](INSTALL.md)           | Insetallation and setup information.               |
| [ARCHITECTURE.md](ARCHITECTURE.md) | System context and design overview                 |
| [PROBLEMS.md](PROBLEMS.md)         | Known limitations and open issues                  |
| [../WARNINGS.md](../WARNINGS.md)   | User-Level Usage Warnings | 
| [GLOSSARY.md](GLOSSARY.md)         | Definitions of terms                               |
| [ADRs.md](ADRs.md)                 | Architectural Decision Records                     |
| [REQUIREMENTS.md](REQUIREMENTS.md) | Project requirements and implementation status     |
| [DATA-MODELS.md](DATA-MODELS.md)   | WindowInfo, DesktopInfo, ScreenInfo dataclasses    |
| [LIBRARY-API.md](LIBRARY-API.md)   | Python API reference                               |
| [CONFIG.md](CONFIG.md)             | Configuration file for all wayland-desktop scripts |
| [SNAPSHOT.md](SNAPSHOT.md)         | Snapshot file format and input_actions DSL         |
| [../ABOUT.md](../ABOUT.md)         | Some background information about this project |
| [dev/README.md](dev/README.md)     | index and annotated notes for all dev scripts      |

---

## Contents

0. [Copyright &amp; License](#cl)

| ADR | Decision |
|---|---|
| [ADR-260317-01](#adr-260317-01) | Use KWin JS scripting engine as the sole window management mechanism |
| [ADR-260317-02](#adr-260317-02) | Use pydbus (GLib/GIO) as the Python D-Bus library |
| [ADR-260317-03](#adr-260317-03) | Use `callDBus()` for KWin-to-Python communication |
| [ADR-260317-04](#adr-260317-04) | Use `workspace.windowAdded` signal for window detection rather than polling |
| [ADR-260317-05](#adr-260317-05) | Detect pre-existing windows by snapshot diff rather than launch timing |
| [ADR-260317-06](#adr-260317-06) | Assign windows to screens via geometry centre-point *(superseded by ADR-260322-01)* |
| [ADR-260317-07](#adr-260317-07) | Assign windows to desktops via `workspace.desktops` object array |
| [ADR-260317-08](#adr-260317-08) | Use YAML for config and snapshot files |
| [ADR-260317-09](#adr-260317-09) | Sort snapshot entries by desktop config order; all-desktops last |
| [ADR-260317-10](#adr-260317-10) | Derive restore commands from `.desktop` file `Exec=` lines |
| [ADR-260317-11](#adr-260317-11) | Assign tracking UUIDs at snapshot time; stamp onto KWin window objects *(superseded by ADR-260321-01)* |
| [ADR-260321-01](#adr-260321-01) | Session state file as the adopted window tracking mechanism |
| [ADR-260321-02](#adr-260321-02) | Atomic move-and-maximize to fix wrong-screen maximization *(superseded by ADR-260322-02)* |
| [ADR-260321-03](#adr-260321-03) | Set onAllDesktops last in place() to prevent geometry drift |
| [ADR-260322-01](#adr-260322-01) | Use workspace.sendClientToScreen() for screen assignment |
| [ADR-260322-02](#adr-260322-02) | Three-phase restore with settle delay |
| [ADR-260322-03](#adr-260322-03) | Fire-and-trust for sendClientToScreen and tile slot read-backs |
| [ADR-260322-04](#adr-260322-04) | Unminimize before sendClientToScreen, re-minimize after |
| [ADR-260322-05](#adr-260322-05) | Unmaximize in phase 1 to clear app self-applied geometry |
| [ADR-260322-06](#adr-260322-06) | Progress window as a spawned child process |
| [ADR-260323-01](#adr-260323-01) | Input injection via XDG RemoteDesktop portal |
| [ADR-260325-01](#adr-260325-01) | Rename restore-usertasks to restore-user-before; add restore-user-after hook |
| [ADR-260325-02](#adr-260325-02) | Return to starting desktop and un-hide progress window at script end and in error handler |

---

### Copyright &amp; License {cl}
Wayland Desktop Restore (documentation) Copyright © 2026 by Sam Azer &lt;sam dot azer at azertech dot net&gt; is licensed under Creative Commons Attribution-ShareAlike 4.0 International. To view a copy of this license, visit https://creativecommons.org/licenses/by-sa/4.0/

#### Creative Commons Attribution-ShareAlike 4.0 International
This license requires that re-users give credit to the creator. It allows re-users to distribute, remix, adapt, and build upon the material in any medium or format, even for commercial purposes. If others remix, adapt, or build upon the material, they must license the modified material under identical terms.

- Credit must be given to the creator.
- Adaptations must be shared under the same terms.

[See the full License](https://creativecommons.org/licenses/by-sa/4.0/)

---

### ADR-260317-01

**Title:** Use KWin JavaScript scripting engine as the sole window management mechanism  
**Date:** 2026-03-17  
**Status:** Accepted

**Context:**  
Window management on KDE Plasma 6 / Wayland requires moving, resizing, and
identifying windows belonging to other processes. The X11 tools previously used
for this (`wmctrl`, `xdotool`) are unavailable. The Wayland protocol explicitly
forbids external processes from manipulating windows they do not own.

**Options considered:**

| Option | Assessment |
|---|---|
| `wmctrl` / `xdotool` | X11-only; do not function on Wayland |
| `qdbus` + KWin D-Bus methods | `qdbus` absent from Plasma 6; `getWindowList()` removed from KWin D-Bus |
| `xdotool`-style Wayland tool (`kdotool`) | Not available/maintained on Tumbleweed at time of evaluation |
| KWin JavaScript scripting via `/Scripting` D-Bus interface | Runs inside compositor; not subject to Wayland restrictions; confirmed available |

**Decision:**  
All window operations are implemented as short-lived JavaScript snippets loaded
into KWin via `org.kde.kwin.Scripting.loadScript()`. Results are returned to
the calling Python script via `callDBus()` callbacks.

**Expected benefit:**  
Full compositor-level access to window objects with no Wayland permission
restrictions. The scripting API is stable, documented, and is the mechanism
KWin itself uses internally for its own window rules.

---

### ADR-260317-02

**Title:** Use pydbus (GLib/GIO) as the Python D-Bus library  
**Date:** 2026-03-17  
**Status:** Accepted

**Context:**  
The library needs to publish a D-Bus service (to receive callbacks from KWin
scripts) and call D-Bus methods (to load/unload scripts, query desktops, etc.).
Three Python D-Bus libraries were available on the system.

**Options considered:**

| Library | Package | Assessment |
|---|---|---|
| `dbus-python` | `python313-dbus-python` | C extension wrapping libdbus directly; returns wrapped types (`dbus.String`, `dbus.Array` etc.) requiring manual conversion; older API |
| `pydbus` | `python313-pydbus` | Built on GLib/GIO via GObject Introspection; returns native Python types automatically; cleaner API; better signal support |
| `dasbus` | Not available | Not in Tumbleweed repos at time of evaluation |
| `jeepney` | Not available | Not in Tumbleweed repos at time of evaluation |

**Decision:**  
Use `pydbus` as the primary library with `dbus-python` as a documented fallback
(for environments where `pydbus` is unavailable). The `KWinBus` class in
`bus.py` supports both via a backend selection mechanism.

**Expected benefit:**  
No manual type conversion required. GLib's `GVariant` unwrapping is handled
automatically. Signal subscription (for `windowAdded`) is more natural. API is
significantly less verbose.

---

### ADR-260317-03

**Title:** Use callDBus() for KWin-to-Python communication rather than file I/O or journal parsing  
**Date:** 2026-03-17  
**Status:** Accepted

**Context:**  
KWin scripts need to return data to the calling Python script. Three mechanisms were evaluated.

**Options considered:**

| Mechanism | Assessment |
|---|---|
| `writeFile()` in KWin JS | Tested; confirmed unavailable — `writeFile is not defined` in this KWin version |
| `print()` to systemd journal, parsed by a Python script | Functional but indirect, fragile, requires `journalctl` subprocess, no clean notification mechanism |
| `callDBus()` to a D-Bus service registered by the Python script | Tested; confirmed working. Direct, event-driven, no polling, no file system dependency |

**Decision:**  
All KWin-to-Python-script communication uses `callDBus()`. The library publishes a small
D-Bus receiver service before loading each script. A GLib main loop in a daemon
thread dispatches incoming method calls. `threading.Event` objects synchronise
the async callbacks with the library's synchronous API.

**Expected benefit:**  
No polling, no files, no journal dependency. The callback arrives immediately
when the script executes. Enables event-driven window detection via the
`windowAdded` signal.

---

### ADR-260317-04

**Title:** Use workspace.windowAdded signal for window detection rather than polling  
**Date:** 2026-03-17  
**Status:** Accepted

**Context:**  
`getWindowList()` was removed from KWin's D-Bus API. After discovering this,
two approaches to detecting new windows after app launch were considered.

**Options considered:**

| Approach | Assessment |
|---|---|
| Poll `getWindowList()` repeatedly | `getWindowList()` does not exist in this KWin version — confirmed by D-Bus introspection |
| Poll `getWindowInfo(uuid)` for known UUIDs | Cannot enumerate UUIDs without a list |
| KWin script hooks `workspace.windowAdded` signal | Fires inside KWin the instant a new window is registered; zero polling latency; confirmed working |

**Decision:**  
The watcher script connects to `workspace.windowAdded` immediately after
sending a snapshot of pre-existing windows. This closes the race condition
where a window could appear between snapshot and signal connection.

**Expected benefit:**  
Sub-100ms detection latency (limited by D-Bus round-trip, not polling interval).
No CPU consumption during the wait. Correct handling of fast-starting apps.

---

### ADR-260317-05

**Title:** Detect pre-existing windows by snapshot diff rather than relying on launch timing  
**Date:** 2026-03-17  
**Status:** Accepted

**Context:**  
When a user runs `kwin_restore.py` without having closed all windows first,
or when an app is already running, the probe must distinguish between
windows that existed before launch and windows created by the new launch.

**Options considered:**

| Approach | Assessment |
|---|---|
| Delay between launch and detection, assume new windows are recent | Fragile; fast apps and slow apps both confuse timing-based approaches |
| Match by PID | Unreliable for Flatpak, Electron, and apps that fork/re-exec |
| Snapshot before launch; any new internalId is the launched window | Reliable; internalId is unique per window per session; no timing dependency |

**Decision:**  
The watcher script sends a snapshot of all open windows immediately on load
(before the app is launched). `probe.py` stores all pre-launch internalIds in a
set. Any `windowAdded` event whose internalId is not in that set is a new
window. If `relaunch=False` and the pre-launch snapshot already contains a
window matching the target `resourceClass`, that window is returned immediately
without launching.

**Expected benefit:**  
Correct behaviour regardless of app startup time. Handles the "already running"
case cleanly. No timing assumptions.

---

### ADR-260317-06

**Title:** Screen assignment via geometry centre-point rather than w.output property  
**Date:** 2026-03-17  
**Status:** Accepted  
**Superseded by:** ADR-260322-01 (2026-03-22) — `workspace.sendClientToScreen()` confirmed as the correct API; geometry approach retired.

**Context:**  
Moving a window to a specific monitor requires assigning it to that screen.
Two approaches were tested.

**Options considered:**

| Approach | Assessment |
|---|---|
| `w.output = screenObject` | Tested; raises `TypeError: Cannot assign to read-only property "output"` |
| `w.sendToOutput(screen)` | Method does not exist on window objects in this KWin version |
| Move window geometry so its centre-point is within target screen rect | Tested; confirmed working. KWin re-associates window with screen containing its centre |

**Decision:**  
`move_to_screen()` positions the window at the top-left corner of the target
screen (preserving width and height). KWin's internal association updates
automatically. A 300ms pause allows KWin to commit the re-association before
subsequent operations.

**Expected benefit:**  
Reliable screen assignment without any unsupported API. The association is
verified by reading `w.output.name` after the move.

**Known issue:**  
If `maximize()` is called immediately after `move_to_screen()`, the maximize
may reference the previous screen before KWin has fully committed the
re-association. This causes windows to expand to the wrong screen size. Fix
pending (see Open Issues §5, issue 2).

---

### ADR-260317-07

**Title:** Desktop assignment uses workspace.desktops object array rather than integer numbers  
**Date:** 2026-03-17  
**Status:** Accepted

**Context:**  
Virtual desktops in KWin Plasma 6 are identified by UUID strings internally,
not by the familiar 1-based integer numbers. The assignment API requires
objects, not integers or strings.

**Options considered:**

| Approach | Assessment |
|---|---|
| `w.desktops = [desktopNumber]` | Rejected; KWin expects desktop objects, not integers |
| `w.desktops = [desktopUuidString]` | Rejected; KWin expects live QObject references, not strings |
| Find desktop object in `workspace.desktops` array matching target UUID, assign | Tested; confirmed working |

**Decision:**  
`DesktopManager.list_desktops()` reads the authoritative desktop list from
`VirtualDesktopManager.desktops` D-Bus property (which returns
`(x11_number, uuid_string, name)` tuples). `move_to_desktop()` scripts iterate
`workspace.desktops` to find the matching object by UUID, then assigns
`w.desktops = [found_object]`.

**Expected benefit:**  
Correct assignment regardless of desktop numbering. UUID-based matching is
stable even if desktop order is rearranged.

---

### ADR-260317-08

**Title:** Use YAML for config and snapshot files  
**Date:** 2026-03-17  
**Status:** Accepted

**Context:**  
Both the user configuration (aliases, app overrides) and the layout snapshot
files are hand-edited by users. A human-readable, comment-supporting format
is required.

**Options considered:**

| Format | Assessment |
|---|---|
| JSON | No comments; unfriendly to hand-edit |
| TOML | Clean syntax; supports comments; requires `python313-tomllib` (read-only) or `tomli-w` (write) |
| YAML | Supports comments; `python313-PyYAML` is a standard Tumbleweed package; familiar to sysadmins |
| INI/ConfigParser | Too limited for nested structures |

**Decision:**  
YAML with `python313-PyYAML`. The snapshot writer uses a custom line-by-line
formatter (not `yaml.dump()` directly) to produce grouped, commented output
with desktop section headings and consistent spacing.

**Expected benefit:**  
Config files are readable and editable without knowledge of the library. Section
headings (`# | Plumbing |`) make the snapshot self-documenting. Comments
explain restore behaviour inline.

---

### ADR-260317-09

**Title:** Snapshot entries sorted by desktop in config order; all-desktops windows last  
**Date:** 2026-03-17  
**Status:** Accepted

**Context:**  
A snapshot may contain windows from many different desktops in arbitrary order.
The restore script processes entries sequentially, so ordering affects
readability and predictability of the restore.

**Options considered:**

| Ordering | Assessment |
|---|---|
| Capture order (KWin windowList order) | Arbitrary; groups nothing; hard to read or edit |
| Alphabetical by desktop name | Ignores user's intentional config ordering |
| Config file ordering | Matches the user's mental model; consistent between captures |

**Decision:**  
Entries are sorted by the position of their desktop in `config.yaml`'s
`desktops:` section. Desktops not in config sort after known ones
alphabetically. Windows with `on_all_desktops=true` use `desktop: '*'` and
sort last under an `# | All desktops |` heading. `on_all_desktops` is removed
as a separate field — `desktop: '*'` is the sole representation.

**Expected benefit:**  
Snapshot files read in the same order as the user thinks about their desktops.
Easy to find and edit entries for a specific desktop. Old snapshots using
`on_all_desktops: true` are still loaded correctly via `from_dict()` migration
logic.

---

### ADR-260317-10

**Title:** Derive restore commands from .desktop file Exec= lines  
**Date:** 2026-03-17  
**Status:** Accepted

**Context:**  
The restore script needs a shell command to launch each application. Apps
launched from the Plasma application menu may have non-obvious CLI invocations.

**Options considered:**

| Source | Assessment |
|---|---|
| `resourceClass` as bare command | Works for many apps; fails for apps whose binary name differs from class (e.g. `thunderbird-esr` → binary `thunderbird`) |
| Ask user to fill in commands manually | Requires editing every snapshot; poor UX |
| Parse `/proc/<pid>/cmdline` at snapshot time | Captures the exact invocation but may include session-specific args; fragile across reboots |
| Read `Exec=` from `.desktop` file | Standard freedesktop mechanism; covers apps installed via packages or Flatpak; reliable |

**Decision:**  
`resolve_command()` in `snapshot.py` tries in order: (1) explicit override in
config `apps` section, (2) `Exec=` from `.desktop` file named by
`desktopFileName`, (3) `Exec=` from `.desktop` file named by `resourceClass`,
(4) bare `resourceClass` as fallback. The `Exec=` line is cleaned of field
codes (`%f`, `%u`, etc.) and `env VAR=value` prefixes per the freedesktop spec.

**Expected benefit:**  
Most apps restore with correct commands automatically. The config override
handles edge cases. The `command` field in the YAML is always user-editable
as a last resort.

---

### ADR-260317-11

**Title:** Assign a tracking UUID to each snapshot entry; stamp onto KWin window object  
**Date:** 2026-03-17  
**Status:** Superseded by ADR-260321-01 — primary approach (Qt dynamic property) confirmed unavailable

**Context:**  
After restore, windows may drift to different screens following an extended
idle period and screen unlock. A realign utility needs to re-place these
windows without running a full restore. This requires a stable handle that
survives within the session but is not the `internalId` (which changes between
reboots) and not a title/class match (which is ambiguous for multi-instance
apps).

**Options considered:**

| Approach | Assessment |
|---|---|
| Match by `resourceClass` + current geometry | Ambiguous when multiple windows of same class exist (e.g. multiple Konsole) |
| External state file mapping tracking UUID → last-known internalId | Requires maintaining state file; adds complexity; file can go stale |
| Qt dynamic property on the window QObject via `w.setProperty("kwinlib_uuid", uuid)` | If available: persists for session lifetime on the QObject; no external state; queryable by scanning `workspace.windowList()` |

**Original decision:**  
A `tracking_uuid` (UUID4) is generated at snapshot capture time and stored in
the YAML. At restore time, after `place()` succeeds, `stamp_tracking_uuid()`
writes the UUID as a Qt dynamic property `"kwinlib_uuid"` on the window object.

**Outcome (2026-03-21):**  
`kwin_property_test.py` was run against a live `KWin::XdgToplevelWindow` object
(Thunderbird, `internalId: 37c36018-2fcd-43c4-ac2e-b079755c876e`).  All three
tests failed: `TypeError: Property 'setProperty' of object
KWin::XdgToplevelWindow(...) is not a function`.  The window was found
correctly — the failure is not a lookup error.  `setProperty` and `property`
are `QObject` methods that exist in C++ but are deliberately not exposed by the
KWin JS sandbox wrapper.  This result is definitive.

A C++ KWin Effect Plugin approach was investigated as an alternative; see
ADR-260321-01 for the full evaluation and the adopted replacement design.

---

### ADR-260321-01

**Title:** Session state file as the adopted window tracking mechanism  
**Date:** 2026-03-21  
**Status:** Accepted

**Context:**  
ADR-260317-11 assumed Qt dynamic properties (`setProperty`/`property`) would
be available in the KWin JS scripting sandbox. Testing on 2026-03-21 confirmed
they are not. An alternative mechanism is required to allow `kwin_realign.py`
to identify any managed window precisely — including distinguishing between
multiple windows of the same `resourceClass` — without relying on window title
or approximate geometry matching.

Additionally, the project has a stated preference for C++ over Python where
technically justified. This was taken into account when evaluating alternatives.

**Options considered:**

**Option A — Signature-based matching (`resourceClass` + screen + geometry)**

| | |
|---|---|
| Complexity | Trivial — no new mechanism needed |
| Multi-instance precision | Poor — multiple Konsole windows on the same screen are indistinguishable |
| Robustness after drift | Poor — geometry changes are exactly what realign is trying to correct |

Rejected: insufficient precision for multi-instance applications.

---

**Option B — C++ KWin Effect Plugin**

A KWin Effect Plugin is a shared library (`.so`) loaded into the KWin process
at startup. From inside a plugin, `KWin::EffectWindow::setData(role, value)` /
`data(role)` provide arbitrary per-window storage — the Effect API equivalent
of `setProperty`/`property`. The plugin would expose a small D-Bus interface;
Python would call it exactly as it calls the KWin scripting interface today.

```
kwin_restore.py
    │
    ├─ D-Bus call → kwinlib_effect.so (C++ plugin inside KWin process)
    │       ├─ tagWindow(internalId, uuid)   → effectWindow->setData(ROLE, uuid)
    │       ├─ getTag(internalId)            → effectWindow->data(ROLE)
    │       └─ findByTag(uuid)              → scan all EffectWindows
    │
    └─ receives result
```

| | |
|---|---|
| Precision | Perfect — tag lives on the window object itself |
| Multi-instance | Perfect |
| Build requirements | `kwin6-devel`, `cmake`, `gcc-c++`, `extra-cmake-modules`, KF6/Qt6 dev headers |
| Installation | `/usr/lib64/qt6/plugins/kwin/effects/`; requires KWin restart to load |
| Maintenance | Must recompile on KWin major API changes |
| Complexity | ~200 lines C++ + CMake; adds a compiled component to a currently pure-Python project |

Deferred, not rejected. The plugin approach is sound and aligns with the
project's C++ preference. It is the recommended path if a future requirement
genuinely needs in-process window tagging. See Future Directions below.

---

**Option C — Session state file (adopted)**

`kwin_restore.py` writes `~/.config/wayland-desktop/session_state.json` at restore
time, mapping each `tracking_uuid` to the `internalId` of the placed window,
along with metadata (`app_alias`, `resource_class`, `desktop`, `screen`,
`placed_at` timestamp).

`kwin_realign.py` reads this file, obtains the `internalId` for each tracked
window, and scans `workspace.windowList()` in a single KWin script to find
and re-place any that have drifted. Entries whose `internalId` is no longer
present are stale (window was closed) and are silently skipped.

| | |
|---|---|
| Precision | Perfect — one entry per window, keyed by UUID |
| Multi-instance | Perfect — each Konsole window gets its own UUID and internalId |
| Stale entries | Silent skip — missing internalId means window was closed |
| Re-restore | File updated automatically on each restore run |
| No compiled component | Pure Python; no build toolchain required |
| Complexity | Trivial — JSON read/write in Python |

**Decision:**  
Option C (session state file) is adopted for the current implementation.
Option B (C++ Effect Plugin) is recorded as the preferred future path if
in-process tagging becomes necessary.

**Consequences:**  
- `kwin_restore.py` writes `session_state.json` after each successful placement.
- `kwin_realign.py` reads `session_state.json` to locate windows by UUID.
- The state file is session-scoped: valid only until the next reboot.
  `placed_at` timestamps allow stale entries from previous sessions to be
  detected and pruned on startup.

**Future directions — C++ Effect Plugin:**  
The plugin approach (Option B) should be revisited if any of the following
arise:
- A third-party tool needs to read window tags set by kwin_lib.
- `kwin_restore.py` cannot be guaranteed to run before the realign utility.
- The KWin Effect API exposes additional capabilities worth exploiting.
- Input injection (REQ-09.4) is required — the same plugin can serve both
  window tagging and input injection in a single build/install step.

Build prerequisites: `sudo zypper install kwin6-devel cmake gcc-c++
extra-cmake-modules`. The plugin skeleton is ~200 lines of C++ and the D-Bus
bridge pattern is well-established in the KDE ecosystem.

---

### ADR-260321-02

**Title:** Atomic move-and-maximize to fix wrong-screen maximization  
**Date:** 2026-03-21  
**Status:** Accepted  
**Superseded by:** ADR-260322-02 (2026-03-22) — the three-phase restore approach
replaces the atomic method; `move_to_screen_and_maximize()` has been removed.

**Context:**  
`kwin_restore.py` was observed maximizing Brave and Dolphin windows to the
dimensions of the *previous* screen rather than the target screen. The
`place()` method was calling `move_to_screen()` and then `maximize()` as two
separate `loadScript()` calls.

**Root cause:**  
`move_to_screen()` includes a 300ms spinloop *inside its JS execution* to
allow KWin to commit the screen re-association before the script returns.
However, once Python receives the callback from that script, it must perform
a full D-Bus round-trip to load and execute the maximize script. During that
round-trip — Python callback handling → `loadScript()` → KWin script queue →
execution — KWin may still be processing the screen re-association internally
when the maximize script executed. `setMaximize()` operates relative to
`w.output`; if `w.output` had not yet updated, the maximize was applied to
the previous screen's dimensions.

**Decision:**  
Add `move_to_screen_and_maximize()` to `WindowManager`. This method performs
both operations inside a single `loadScript()` call: move the window, spin for
300ms, then call `setMaximize(true, true)` — all within one JS execution.
`place()` calls this method when both screen assignment and maximization are
required. `move_to_screen()` is retained for use when only repositioning is
needed (e.g. for minimized windows).

**Benefit:**  
Eliminates the inter-call timing gap entirely. The maximize is guaranteed to
fire after the in-script re-association pause, with `w.output` already updated
to the target screen.

---

### ADR-260321-03

**Title:** Set onAllDesktops last in place() to prevent geometry drift  
**Date:** 2026-03-21  
**Status:** Accepted — fix applied; under observation

**Context:**  
Windows set to appear on all virtual desktops (`desktop: '*'`) were changing
size and position when the user switched between desktops after restore.

**Root cause (hypothesis):**  
`set_on_all_desktops()` was being called in step 2 of `place()`, before the
screen move and maximize. Setting `w.onAllDesktops = true` causes KWin to
propagate the window to every virtual desktop and trigger per-desktop geometry
negotiation with the application. KWin stores the window's geometry at the
moment of propagation as the per-desktop reference state. When maximize fired
afterward, it would win initially, but KWin had already stored pre-maximize
per-desktop geometry; switching desktops caused KWin to re-apply that stored
state, undoing the maximize.

**Decision:**  
Move `set_on_all_desktops()` to the last step of the finalise phase, after all
geometry and state operations (screen move, maximize) are complete. KWin then
propagates the window to all desktops starting from its final correct state
rather than an intermediate one.

**Observation note:**  
This fix is based on the above hypothesis rather than a confirmed root-cause
trace. If geometry drift recurs in testing, a more aggressive fallback would
be to re-apply geometry on each desktop after setting the flag — but this is
disruptive and deferred unless the current fix proves insufficient.

---

### ADR-260322-01

**Title:** Use workspace.sendClientToScreen() for screen assignment  
**Date:** 2026-03-22  
**Status:** Accepted — supersedes ADR-260317-06

**Context:**  
ADR-260317-06 adopted a geometry centre-point approach to screen assignment
because `w.output` was read-only and `sendToOutput()` did not exist. This
approach worked but had a documented limitation: the 300ms in-script pause was
not always sufficient, leading to the atomic `move_to_screen_and_maximize()`
workaround (ADR-260321-02).

Further testing on 2026-03-22 (Test N) discovered that
`workspace.sendClientToScreen(window, output)` exists and works correctly in
this KWin version. This API is the proper compositor-level screen assignment
call, requiring no geometry tricks.

**Decision:**  
Replace the geometry centre-point approach in `move_to_screen()` with
`workspace.sendClientToScreen(__w, targetOutput)` where `targetOutput` is
found by iterating `workspace.screens` by connector name. The geometry
positioning code is removed entirely.

**Constraint discovered:**  
`sendClientToScreen()` silently ignores minimized windows — the window has no
on-screen geometry so the compositor cannot relocate it. The caller must
unminimize before calling and re-minimize afterward if needed. See ADR-260322-04.

**Read-back not possible:**  
`__w.output.name` does not update within the same script execution after
`sendClientToScreen()` is called. See ADR-260322-03.

---

### ADR-260322-02

**Title:** Three-phase restore with settle delay  
**Date:** 2026-03-22  
**Status:** Accepted — supersedes ADR-260321-02

**Context:**  
ADR-260321-02 introduced `move_to_screen_and_maximize()` as an atomic method
to avoid the inter-call timing gap between screen move and maximize. With the
adoption of `sendClientToScreen()` (ADR-260322-01) and the discovery that
minimized windows are ignored by `sendClientToScreen()` (ADR-260322-04), the
single-phase approach needed further redesign.

The core problem is that KWin commits screen re-associations asynchronously.
Any state operation (maximize, tile) that follows a screen move must wait for
that commitment. A fixed in-script delay is not reliable because the commitment
time varies.

**Decision:**  
The restore is restructured into three phases separated by a configurable
settle delay (default 3 seconds):

**Phase 1 — Launch and identify:**  
Each app is launched, its window identified, and immediately `unmaximize()`
then `minimize()` called. The unmaximize clears app self-applied geometry
(see ADR-260322-05). The minimize parks the window.

**Phase 2 — Move:**  
Each window is `unminimize()`d, moved to target desktop and screen, then
`minimize()`d. The unminimize/re-minimize cycle is required by ADR-260322-04.

**Settle delay:**  
A configurable pause (default 3 seconds) between phases 2 and 3. During this
time KWin commits all screen re-associations for all windows simultaneously.

**Phase 3 — Finalise:**  
Each window is `unminimize()`d, `unmaximize()`d (clears any geometry
re-applied between phases), then the target state is applied (maximize / tile
/ minimize / raise), then `set_on_all_desktops()` last (ADR-260321-03).

**Benefit:**  
The 3-second settle delay is conservative but eliminates all timing-related
placement failures. Phase 3 geometry operations are guaranteed to fire after
all screen re-associations are complete. The `move_to_screen_and_maximize()`
atomic method and `place()` are no longer needed and have been removed.

---

### ADR-260322-03

**Title:** Fire-and-trust for sendClientToScreen and tile slot read-backs  
**Date:** 2026-03-22  
**Status:** Accepted

**Context:**  
During restore testing, `move_to_screen()` attempted to verify the screen
assignment by reading `__w.output.name` within the same script execution
after calling `sendClientToScreen()`. The read-back consistently returned
the pre-move value, causing all moves to be reported as failures even though
the windows arrived on the correct screens.

The same stale-read behaviour had already been documented for tile slot methods
(`slotWindowQuickTileLeft` etc.) in `kwin_tile_test.md`.

**Root cause:**  
KWin processes screen re-association asynchronously after `sendClientToScreen()`
returns. Within the same script execution, `__w.output` still reflects the
window's previous screen. The updated value only becomes visible to a
subsequent, separate script load.

**Decision:**  
`move_to_screen()` and `tile()` are fire-and-trust: the JS calls the API and
immediately sends `{ ok: true }` without reading back the result. The only
error path is "screen/slot name not found", which is a genuine configuration
error correctly raising `RuntimeError` in Python.

Correctness is verified indirectly: in phase 3, `get_screen_geometry()` runs
in a new script load after the settle delay and reflects the true post-move
state. The `get_screen_geometry()` read-back IS reliable because it runs in a
new script load, not the same one that called `sendClientToScreen()`.

---

### ADR-260322-04

**Title:** Unminimize before sendClientToScreen in phase 2, re-minimize after  
**Date:** 2026-03-22  
**Status:** Accepted

**Context:**  
After implementing ADR-260322-01 (`sendClientToScreen()`), phase 2 of the
restore still reported 16 out of 20 moves failing. The log showed all failures
as the window remaining on `HDMI-A-2` (the KWin default screen for new
windows) regardless of the target screen. Verbose output confirmed the
`move_to_screen` script was completing without error.

**Root cause confirmed:**  
`workspace.sendClientToScreen()` silently ignores minimized windows. The window
has no on-screen geometry, so the compositor has nothing to relocate across
outputs. The call returns without error but does nothing.

**Decision:**  
In phase 2 `move_entry()`, wrap the move with unminimize/re-minimize:
1. `unminimize()` — gives the window on-screen geometry
2. `move_to_desktop()` (if applicable)
3. `move_to_screen()`
4. `minimize()` — re-parks the window for the settle delay

Both the unminimize and re-minimize are in `try/except` so a failure in either
is non-fatal — the move still proceeds.

**Verification:**  
After this fix, all 20 windows in a full restore moved successfully in a single
run with zero failures.

---

### ADR-260322-05

**Title:** Unmaximize in phase 1 to clear app self-applied geometry  
**Date:** 2026-03-22  
**Status:** Accepted

**Context:**  
After a successful full restore, Brave Browser's window was functional but
arrived with geometry 1920×1032 (full screen width, height minus taskbar)
that it had self-applied during its own asynchronous session restore. KWin
did not report the window as maximized (`maximized: False` in verbose output),
but the window refused normal drag-to-move because its width exactly equalled
the screen width, triggering KWin's edge-snap logic.

Multiple attempts to clear this geometry in phase 3 failed: calling
`unmaximize()` (which sends `setMaximize(false, false)`) in phase 3
acknowledged the request (`maximized: False` in read-back) but the geometry
did not change — Brave's own session restore was still running asynchronously
and re-asserting its preferred size faster than our configure events could
clear it.

**Decision:**  
In phase 1, immediately after identifying each window, call `unmaximize()`
before `minimize()`. This sends `setMaximize(false, false)` very early —
giving the app the entire duration of phase 1 (remaining windows), phase 2
(all moves), and the 3-second settle delay to process the configure event and
relinquish its self-applied geometry before phase 3 touches it.

A second `unmaximize()` call is retained at the start of phase 3 to clear any
geometry the app may have re-applied between the phase 1 call and phase 3.

**Outcome:**  
Brave's window is placed correctly and is freely moveable after restore. The
phase 1 timing gives the app 30+ seconds of lead time, which is sufficient for
all tested apps.

---

### ADR-260322-06

**Title:** Progress window implemented as a spawned child process  
**Date:** 2026-03-22  
**Status:** Accepted

**Context:**  
`kwin_restore.py` runs synchronously on the main thread. A Qt GUI requires
its own event loop. The options were a background thread with Qt event loop,
or a separate process.

Qt is not thread-safe in general: widgets must be created and manipulated on
the thread that owns the `QApplication` instance. Running a Qt event loop in
a background thread of a non-Qt Python process is technically possible but
fragile and unsupported by PyQt6. Using `multiprocessing.fork()` is also
unsafe with Qt (fork+exec is required).

**Decision:**  
The progress window runs in a child process started with
`multiprocessing.get_context("spawn")`. Communication uses three lightweight
shared primitives:

- `cmd_queue` (`multiprocessing.Queue`): parent → child log messages and
  completion signal
- `verbose_val` (`multiprocessing.Value('b')`): child → parent, Verbose
  checkbox state, read by restore loops on each per-window iteration
- `abort_val` (`multiprocessing.Value('b')`): child → parent, Abort button
  state, checked at the top of each per-window iteration

A `QTimer` at 250ms (4 updates/second) drains `cmd_queue` in the child and
appends colour-coded lines to the log pane. Window geometry is persisted to
`~/.config/wayland-desktop/progress_window.json` on close and restored on next launch.

`ProgressWindowLogger` wraps `ProgressWindow` and mirrors every message to
both the terminal and the window, with no `if win:` guards required at call
sites. When `win=None` (window disabled), it writes to the terminal only.

`kwin_restore.py` blocks on `win.wait_for_close()` after calling
`win.set_complete()`, so the terminal prompt does not return until the user
dismisses the window. This ensures clean process cleanup and avoids orphaned
child processes.

---

*End of ADR log. New decisions should be added with sequential numbers on
their date, e.g. ADR-260401-01 for the first decision taken on 2026-04-01.*

---

### ADR-260323-01 — Input injection via XDG RemoteDesktop portal

**Date:** 2026-03-23  
**Status:** Accepted

**Context:**  
Phase 4 requires injecting keystrokes and mouse events into placed windows.
Three mechanisms were investigated:

1. `org.kde.KWin.EIS.RemoteDesktop` (KWin private interface) — grants
   `POINTER_ABSOLUTE`, `TOUCH`, `BUTTON`, `SCROLL` only. The seat capability
   inspection confirmed `caps=['TOUCH']`; `KEYBOARD` is deliberately withheld
   by KWin's security policy regardless of requested capabilities.

2. `org.freedesktop.portal.RemoteDesktop` (XDG portal) — grants all
   capabilities including `KEYBOARD` after a one-time user authorization
   dialog. Provides `NotifyKeyboardKeycode`, `NotifyPointerMotionAbsolute`,
   `NotifyPointerButton`, `NotifyPointerAxisDiscrete` as direct D-Bus calls.
   No libei / ctypes required.

3. ydotool / C++ KWin plugin — deferred; not needed given option 2 works.

**Findings during implementation:**

- pydbus returns the raw fd list index (0) for `QDBusUnixFileDescriptor`
  return values; `Gio.DBusConnection.call_with_unix_fd_list_sync()` is
  required to extract the actual file descriptor.
- `ei_seat_bind_capabilities` is a variadic C function. Without explicit
  `argtypes` the first argument (pointer) was truncated, causing a segfault.
  Declaring full argtypes for the fixed 4-capability call resolved this.
- KWin's private EIS interface seat advertises `caps=['TOUCH']` only —
  confirmed by `ei_seat_has_capability` probe.
- All three portal request methods (CreateSession, SelectDevices, Start) are
  async: each returns a request object path immediately and fires a `Response`
  signal when complete. A GLib main loop is required to receive the signal.
- Gio `signal_subscribe` callbacks take 7 arguments, not 6; the last is
  `user_data` passed at subscription time.
- `GLib.Variant("a{sv}", {})` passed inside another Variant constructor
  produces type `v` (wrapped variant), not `a{sv}` (inline dict). All
  `Notify*` calls must pass `{}` directly as a Python dict and let PyGObject
  infer the `a{sv}` type from the outer format string.

**Decision:**  
Use `org.freedesktop.portal.RemoteDesktop` exclusively for all input
injection. The one-time authorization dialog (`persist_mode=2`) issues a
restore token saved to `~/.config/wayland-desktop/portal_session_token`; subsequent
runs restore the session silently with no dialog. All four injection methods
(keyboard, pointer move, button, scroll) are pure D-Bus calls via
`Gio.DBusConnection.call_sync()`. libei and ctypes are not used.

The portal session lifecycle (CreateSession → SelectDevices → Start →
Notify* calls → Close) is encapsulated in `InputInjector` in
`kwin_lib/input_injection.py`, used as a context manager by the phase 4
loop in `kwin_restore.py`.

---

## ADR-260325-01

**Title:** Rename `restore-usertasks` to `restore-user-before`; add `restore-user-after` hook  
**Date:** 2026-03-25  
**Status:** Accepted

**Context:**  
The `restore` bash wrapper supported one optional user hook script,
`restore-usertasks`, executed before `kwin_restore.py` launched.  Users
needed a symmetric post-restore hook — for tasks such as opening a terminal,
sending a desktop notification, or cleaning up temporary state set up by the
pre-restore hook — but no such hook existed.

The name `restore-usertasks` was also ambiguous: it did not indicate *when*
in the sequence it ran, making the naming convention opaque as soon as a
second hook was added.

**Decision:**  
1. Rename `restore-usertasks` → `restore-user-before` throughout the codebase
   and all documentation.  The semantics are unchanged: it runs after the
   confirmation dialogs, before `kwin_restore.py` is invoked.
2. Add a new `restore-user-after` hook, looked up in the same
   `user_scripts_folder` directory.  It is invoked after `kwin_restore.py`
   returns, **regardless of whether `kwin_restore.py` succeeded or failed**.
   The `restore` wrapper captures `kwin_restore.py`'s exit code, runs the
   hook, then re-applies the exit code.
3. Because `exec python3 ...` replaced the shell process and prevented any
   code after it from running, the final `exec` is replaced with a plain
   `python3 ...` call and an explicit `exit "$_restore_exit"`.  The `--dry-run`
   path retains `exec` since no post-hook is needed there.

**Consequences:**  
- Any user who created a `restore-usertasks` script must rename it to
  `restore-user-before`.  No change to the script's contents is required.
- The new naming scheme (`restore-user-before` / `restore-user-after`) makes
  the execution order self-documenting.
- `restore-user-after` enables post-restore automation without requiring users
  to modify `kwin_restore.py` or wrap the `restore` script themselves.

---

## ADR-260325-02

**Title:** Return to starting desktop and un-hide progress window at script end and in error handler  
**Date:** 2026-03-25  
**Status:** Accepted

**Context:**  
`kwin_restore.py` already switched back to the originally active desktop
between phase 3 and phase 4 (input actions).  That switch ensured a known
desktop state before input actions fired, but it ran *before* phase 4, not
after it.  Phase 4 input actions can themselves switch desktops (e.g. to
interact with a window on a different virtual desktop).  After all phases
completed, the user was left on whatever desktop phase 4 happened to end on.

Additionally, phase 4 hides the progress window around each input action
entry.  An exception or an abort inside phase 4 could exit before the
corresponding `show_window()` call, leaving the progress window hidden when
the session error message was reported.

**Decision:**  
1. At the very end of the `with KWinSession(...)` block (after phase 5, after
   all phases have completed), call `logger.show_window()` unconditionally to
   guarantee the progress window is visible regardless of what phase 4 did.
   Then switch back to `restore_desktop_id` using the same pattern as the
   existing pre-phase-4 switch.  The pre-phase-4 switch is retained unchanged;
   the new end-of-script switch is in addition to it.
2. In the `except RuntimeError` error handler, call `logger.show_window()`
   before reporting the error.  Then attempt a best-effort desktop switch
   using a fresh `KWinSession` (the session inside the `with`-block may be
   closed at this point).  The fresh session is opened inside its own
   `try/except Exception: pass` so any failure is silently swallowed — the
   primary responsibility of the error handler is to surface the error to the
   user, not to guarantee a desktop switch.
3. `restore_desktop_id` is initialised to `None` *before* the outer `try:`
   block so that both the `except` and `finally` handlers can reference it
   safely even if the exception fires before the variable is first assigned
   inside the `with`-block.

**Consequences:**  
- After a successful run the user always lands back on the desktop they were
  on when they invoked `restore`, regardless of what phase 4 did.
- After a session error the progress window is guaranteed to be visible, and
  a best-effort desktop switch is attempted before the error message is shown.
- The pre-phase-4 desktop switch (ensuring a known state before input actions)
  is preserved and unchanged; the new switch is purely additive.

## ADR-260329-01

**Title:** Change `progress_window` config setting from boolean to screen alias string

**Status:** Accepted

**Context:**  
The original `progress_window: true/false` setting could only show or hide
the progress window.  It offered no control over which screen the window
appeared on, which meant it was frequently hidden behind other windows on
the wrong screen during a multi-monitor restore run.  Additionally, the
window was set `keepAbove` (always on top) even though the user's intent
was simply to keep it accessible — not to have it obscure the windows
being restored.

**Decision:**  
Replace the boolean `progress_window` with a screen alias string:

- **Blank string `""`** — progress window disabled (default).
- **Screen alias** — progress window enabled; the window is moved to the
  named screen and `keepBelow` is engaged so it stays behind other windows
  during the restore.

Validation of the alias is performed at startup (before any windows are
moved), so a misconfigured alias causes an immediate, clear error rather
than a silent misfiring at runtime.

The `--show-window` CLI flag continues to force the window on.  When
`progress_window` contains a valid alias, `--show-window` also inherits
the screen placement and `keepBelow` behaviour.  When `progress_window` is
blank and only `--show-window` is passed, the window opens wherever KWin
chooses with no `keepBelow`.

**Lifecycle changes introduced alongside this ADR:**

| Event | Action |
|---|---|
| After `win.show()` (alias configured) | Move to screen; set `keepBelow=true` |
| End of Phase 4 | Minimize the window |
| End of Phase 5 | Clear `keepBelow`, unminimize, raise to front |
| Error path | Clear `keepBelow`, unminimize, raise to front |

**Geometry save/restore removed:**  
The previous implementation saved and restored the window's x/y/w/h to
`~/.config/wayland-desktop/progress_window.json`.  This was removed because
explicit screen placement supersedes saved geometry, and persisted geometry
was a source of confusing placement when the screen layout changed.

**Consequences:**  
- Users who previously set `progress_window: true` must update their config
  to `progress_window: "screen-alias"` (or use `--show-window` on the CLI).
- The progress window is now guaranteed to be visible at the end of a
  successful run, having been raised out from behind other windows.
- `keepAbove` is no longer set at any point; the window will not obscure
  the windows being restored.
