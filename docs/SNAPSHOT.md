<!-- SPDX-FileCopyrightText: 2026 Sam Azer <sam dot azer at azertech dot net>  -->
<!-- SPDX-License-Identifier: CC-BY-SA-4.0  -->
# wayland-desktop — Snapshot File Reference

A snapshot file is a YAML file produced by `snapshot` and consumed
by `restore`.  It records one window per entry under the `windows:`
key, grouped by virtual desktop.

This document describes every field and the `input_actions` DSL in full.

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
- [File structure](#file-structure)
- [Field reference](#field-reference)
- [input_actions DSL](#input_actions-dsl)
  - [Plain text](#plain-text)
  - [Key tokens](#key-tokens)
  - [Modifier aliases](#modifier-aliases)
  - [Mouse tokens](#mouse-tokens)
  - [Control tokens](#control-tokens)
  - [Escaping](#escaping)
  - [YAML long-string syntax](#yaml-long-string-syntax)
  - [Examples](#examples)

---

### Copyright &amp; License
Wayland Desktop Restore (documentation) Copyright © 2026 by Sam Azer &lt;sam dot azer at azertech dot net&gt; is licensed under Creative Commons Attribution-ShareAlike 4.0 International. To view a copy of this license, visit https://creativecommons.org/licenses/by-sa/4.0/

#### Creative Commons Attribution-ShareAlike 4.0 International
This license requires that re-users give credit to the creator. It allows re-users to distribute, remix, adapt, and build upon the material in any medium or format, even for commercial purposes. If others remix, adapt, or build upon the material, they must license the modified material under identical terms.

- Credit must be given to the creator.
- Adaptations must be shared under the same terms.

[See the full License](https://creativecommons.org/licenses/by-sa/4.0/)

---

## File structure

```yaml
# =============================================================================
# KWin desktop layout snapshot
# Captured: 2026-03-17T23:25:17  host: workstation.localhost
# ...
# =============================================================================

meta:
  timestamp: '2026-03-17T23:25:17'
  hostname:  'workstation.localhost'

windows:

  # =============================
  # | Main  (Main) |
  # =============================

  - app: thunderbird
    resource_class: thunderbird-esr
    desktop: Main
    screen: main
    ...

  # =============================
  # | All desktops |
  # =============================

  - app: google-chrome
    desktop: '*'
    ...
```

Entries within each desktop group are sorted alphabetically by `app` alias.
The `All desktops` group always appears last.

---

## Field reference

### Identity fields

| Field | Type | Written by | Description |
|---|---|---|---|
| `app` | string | snapshot | Friendly alias from `config.yaml`, or `resource_class` if no alias is defined. Used as the human-readable label throughout the tool. |
| `resource_class` | string | snapshot | The KWin `resourceClass` string (e.g. `thunderbird-esr`, `org.kde.konsole`). Used to identify and launch the window. |
| `desktop_filename` | string | snapshot | Base name of the application's `.desktop` file (e.g. `thunderbird`). Used to derive the launch command. |

### Placement fields

| Field | Type | Written by | Description |
|---|---|---|---|
| `desktop` | string | snapshot | Virtual desktop name or alias (e.g. `Main`, `Job 1`). Use `'*'` to place the window on all virtual desktops simultaneously. |
| `screen` | string | snapshot | Screen alias or connector name (e.g. `main`, `HDMI-A-2`). Aliases are defined in `config.yaml`. |

### State fields

| Field | Type | Written by | Description |
|---|---|---|---|
| `maximized` | bool | snapshot | Whether the window was maximized at capture time. |
| `minimized` | bool | snapshot | Whether the window was minimized at capture time. Rarely true in practice. |
| `tile` | string | snapshot/hand | Snap position instead of maximizing. Values: `left`, `right`, `top`, `bottom`, `top-left`, `top-right`, `bottom-left`, `bottom-right`. Empty string means maximize (if `maximized: true`) or raise. |

### Geometry fields

| Field | Type | Written by | Description |
|---|---|---|---|
| `x` | int | snapshot | Window left edge in compositor coordinates at capture time. |
| `y` | int | snapshot | Window top edge in compositor coordinates at capture time. |
| `width` | int | snapshot | Window width at capture time. |
| `height` | int | snapshot | Window height at capture time. |

Geometry is recorded for reference. On Wayland, `restore` cannot
force a window to a specific size — only maximize and tile operations are
reliable for size management.

### Restoration fields

| Field | Type | Written by | Description |
|---|---|---|---|
| `command` | string | snapshot | Shell command used to launch the app (derived from `.desktop` Exec= line). Edit this if an app doesn't launch correctly, or if you want to open a specific file or URL. |
| `single_instance` | bool | snapshot/hand | If `true`, skip launching if a window with this `resource_class` is already open. Useful for apps like Thunderbird that should only have one instance. |
| `ignore_window` | bool | snapshot/hand | If `true`, launch the app without waiting for a window (e.g. KeePassXC, which opens only to the system tray). No placement is performed. |

### Session tracking

| Field | Type | Written by | Description |
|---|---|---|---|
| `tracking_uuid` | string | snapshot | UUID assigned at capture time. Used by `realign` to find this window by UUID scan rather than by class or title. **Do not edit.** If absent from an old snapshot file, a new UUID is generated automatically on first load. |

### Input actions

| Field | Type | Written by | Description |
|---|---|---|---|
| `input_actions` | string | **hand only** | DSL string executed against this window in phase 4 of the restore. `snapshot` always writes this as an empty string. Fill it in by hand. See the [input_actions DSL](#input_actions-dsl) section below. |

### Informational fields (read-only)

These fields are written by `snapshot` for reference only.
`restore` ignores them.

| Field | Type | Description |
|---|---|---|
| `_caption` | string | Window title at capture time. |
| `_pid` | int | Process ID at capture time. |

---

## input_actions DSL

The `input_actions` field contains a single string that is parsed as a
sequence of actions and executed against the window after it has been
placed (phase 4 of the restore).

The string mixes **plain text** (typed literally) and **special tokens**
enclosed in `{...}`.  Everything inside `{...}` is case-insensitive.

### Plain text

Any text not inside `{...}` is typed character by character into the
focused window.

```
ssh user@server.localhost
```

Whitespace in plain text is preserved — spaces and tabs are typed as-is.
Newlines in the YAML string are treated as whitespace separators between
tokens and are not typed (use `{ENTER}` to send the Enter key).

**Comment lines:** Any line whose first non-whitespace character is `#`
is silently ignored by the parser.  This allows you to paste the full
output of `mouse-clicks` — including its `#` comment lines —
directly into `input_actions` without editing.  Only the DSL token lines
are executed; the comment lines are discarded.

```yaml
input_actions: |
  # click the File menu
  {WCLICK 45, 22}
  # type the search path and confirm
  /home/sam/docs/{ENTER}
```

### Key tokens

A token containing only a key name (with optional modifiers) sends that
key combination:

```
{ENTER}          Enter / Return key
{ESC}            Escape key
{TAB}            Tab key
{SPACE}          Space bar
{BS}             Backspace
{DEL}            Delete
{INS}            Insert
{HOME}           Home
{END}            End
{UP}             Arrow up
{DOWN}           Arrow down
{LEFT}           Arrow left
{RIGHT}          Arrow right
{PGUP}           Page Up  (also: {PAGEUP})
{PGDN}           Page Down  (also: {PAGEDOWN})
{F1} … {F12}     Function keys
{CAPSLOCK}       Caps Lock toggle
{PRTSC}          Print Screen  (also: {PRINTSCREEN})
```

**Key combinations** use `+` to join modifiers and a key:

```
{CTRL+T}         Ctrl+T  (new tab in most browsers and Konsole)
{CTRL+SHIFT+T}   Ctrl+Shift+T  (reopen closed tab)
{CTRL+L}         Ctrl+L  (focus address bar)
{ALT+F4}         Alt+F4  (close window)
{META+LEFT}      Meta+Left  (tile window left)
{SHIFT+HOME}     Shift+Home  (select to beginning of line)
```

### Modifier aliases

All of the following are accepted and map to the same key:

| You type | Sends |
|---|---|
| `CTRL` | Left Ctrl |
| `SHIFT` | Left Shift |
| `ALT` | Left Alt |
| `META` `SUPER` `WIN` `WINDOWS` `START` `OPT` `OPTION` | Left Meta (Super/Windows key) |

So `{OPT+LEFT}`, `{WIN+LEFT}`, `{START+LEFT}`, and `{META+LEFT}` all send
the same keystroke.

### Key name aliases

Common alternative spellings are all recognised:

| Canonical | Also accepted |
|---|---|
| `ENTER` | `RETURN` |
| `ESC` | `ESCAPE` |
| `BS` | `BACKSPACE` |
| `DEL` | `DELETE` |
| `INS` | `INSERT` |
| `PGUP` | `PAGEUP` `PAGE_UP` `PG_UP` |
| `PGDN` | `PAGEDOWN` `PAGE_DOWN` `PG_DN` `PGDOWN` |
| `PRTSC` | `PRINTSCREEN` `PRINT_SCREEN` |
| `SPACE` | `SPC` |

### Mouse tokens

Mouse actions require the window to have focus.  Use `{FOCUS}` before a
mouse action if focus may have been lost.

The recommended way to capture mouse coordinates is to run
`mouse-clicks` while interacting with the target window — it logs
ready-to-paste DSL tokens with window-relative coordinates for every click.
See `README.md` for usage.

```
{MOVE 1234,567}     Move the mouse pointer to absolute compositor
                    coordinates (x, y).  Required before {CLICK} if
                    you want to click at a specific position.

{CLICK}             Left-click at the current pointer position.

{CLICK 1234,567}    Move to (x, y) then left-click.

{RCLICK 1234,567}   Right-click at (x, y).  Absolute coordinates.

{MCLICK 1234,567}   Middle-click at (x, y).

{WCLICK 100,50}     Left-click at offset (100, 50) within the focused
                    window's top-left corner.  Window-relative — safe
                    to use even if the window moves between capture
                    and replay, as long as the window size is the same.

{WRCLICK 100,50}    Right-click at offset (100, 50) within the focused
                    window's top-left corner.  Window-relative right
                    click — the companion to {WCLICK}.

{WDRAG 100,50 to 300,50}
                    Left-button drag from window-relative offset (100, 50)
                    to (300, 50).  Both coordinates are relative to the
                    window's top-left corner at replay time.  This is the
                    token emitted by mouse-clicks when a left-button
                    drag is detected.

{WRDRAG 100,50 to 300,50}
                    Right-button drag from window-relative offset (100, 50)
                    to (300, 50).  Same coordinate convention as {WDRAG}.

{DRAG 1234,567 to 1390,567}
                    Left-button drag using absolute compositor coordinates.
                    Use {WDRAG} in preference — absolute coordinates are
                    fragile across screen layout changes.

{SCROLLUP}          Scroll up one notch at the current pointer position.
{SCROLLDOWN}        Scroll down one notch.
```

**Note on coordinates:** Absolute coordinates are in the compositor
coordinate space — the same space reported by `get_screen_geometry()`.
If your primary monitor starts at x=3840 (because it is to the right of
another monitor), you must account for that offset.  Window-relative
`{WCLICK}` and `{WRCLICK}` coordinates are always relative to the window's
top-left corner regardless of its screen position, making them more
robust to window moves.

### Control tokens

```
{SLEEP 100ms}       Pause for 100 milliseconds.
{SLEEP 1.5s}        Pause for 1.5 seconds.
{WAIT 500}          Pause for 500 milliseconds (alternative spelling).
{FOCUS}             Re-raise and focus the target window.  Use this if
                    a previous action may have caused focus to move to
                    another window (e.g. after opening a dialog).
{FOCUS-MAIN}        Alias for {FOCUS}.  Use this for clarity when your
                    sequence alternates between the main window and a
                    tool window (e.g. a file dialog).                    
{ACTIVE-WINDOW}     Capture the currently active window as the "tool
                    window".  Place this after a {SLEEP} that gives a
                    just-opened dialog or tool window time to appear and
                    steal focus.  The captured UUID is held for the
                    duration of this entry's input_actions sequence and
                    reset at the start of the next entry.
{FOCUS-TOOL}        Raise and focus the tool window captured by the most
                    recent {ACTIVE-WINDOW} token.  If no tool window has
                    been captured yet, falls back to the main window and
                    logs a warning.
{PASS key}          Look up the Secret Service item whose label exactly
                    matches *key* and type the returned secret as plain
                    text into the focused window.  No Enter key is
                    appended — add {ENTER} explicitly if needed.

                    Requires use_secret_service: true in config.yaml
                    and KeePassXC running with Secret Service Integration
                    enabled (Tools → Settings → Secret Service
                    Integration).  The *key* is the entry title as it
                    appears in KeePassXC (case-sensitive).

                    Example:
                        {PASS router-admin}{ENTER}
                        {PASS user@example.com}
```

Typical tool-window pattern:

```yaml
input_actions: |
  {WCLICK 45, 22}      # click the menu item that opens a file dialog
  {SLEEP 500ms}        # wait for the dialog to appear and become active
  {ACTIVE-WINDOW}      # capture it as the tool window
  {WCLICK 200, 150}    # click inside the file dialog
  {ENTER}              # confirm
  {FOCUS-MAIN}         # return focus to the main window
```

### Escaping

To type a literal `{`, write `{{`.  A literal `}` is just `}` (no escape
needed, since `}` is only special when closing a `{`).

```
Type {{some text}   →  types the literal string "{some text"
```

### YAML long-string syntax

For short action sequences, a single quoted or double-quoted string works:

```yaml
input_actions: "{CTRL+T}{SLEEP 300ms}https://example.com/{ENTER}"
```

For longer sequences, use YAML's **literal block scalar** (`|`).  Lines
in a literal block scalar are joined with newlines in YAML, but the
`input_actions` scanner ignores newlines between tokens — you can wrap
freely for readability:

```yaml
input_actions: |
  {CTRL+T}
  {SLEEP 300ms}
  {CTRL+L}
  https://example.com/
  {ENTER}
```

This produces the same result as the single-line version.  The `|` style
is recommended for sequences of more than two or three tokens.

You can also use the **folded block scalar** (`>`), which folds newlines
to spaces.  The result is identical for the scanner since whitespace
between tokens is ignored:

```yaml
input_actions: >
  {CTRL+T}{SLEEP 300ms}
  {CTRL+L}https://example.com/{ENTER}
```

---

## Examples

### Open a URL in a new Chrome tab

```yaml
- app: google-chrome
  desktop: '*'
  screen: top
  maximized: false
  tile: right
  input_actions: |
    {CTRL+T}
    {SLEEP 300ms}
    {CTRL+L}
    https://mail.google.com/
    {ENTER}
```

### Open multiple tabs in Chrome

```yaml
- app: google-chrome
  desktop: '*'
  screen: top
  maximized: false
  tile: left
  input_actions: |
    {CTRL+T}{SLEEP 200ms}{CTRL+L}https://github.com/{ENTER}
    {SLEEP 500ms}
    {CTRL+T}{SLEEP 200ms}{CTRL+L}https://invent.kde.org/{ENTER}
```

### Set up a Konsole tab with SSH and a running command

```yaml
- app: konsole
  desktop: Plumbing
  screen: right
  maximized: true
  input_actions: |
    {CTRL+SHIFT+T}
    {SLEEP 300ms}
    ssh user@server.localhost{ENTER}
    {SLEEP 1.5s}
    htop{ENTER}
```

### Brave — open a specific URL (Meta+Left tiles, then load a page)

```yaml
- app: brave
  desktop: '*'
  screen: left-bottom
  maximized: false
  input_actions: |
    {META+LEFT}
    {SLEEP 200ms}
    {CTRL+L}
    https://intranet.example.com/
    {ENTER}
```

### Click a button at a known window-relative position

```yaml
  input_actions: |
    {FOCUS}
    {SLEEP 200ms}
    {WCLICK 120,45}
```

### Konsole — multiple tabs with different working directories

```yaml
  input_actions: |
    cd /storage/projects/wayland-desktop{ENTER}
    {CTRL+SHIFT+T}{SLEEP 300ms}
    cd /storage/home-config{ENTER}
    {CTRL+SHIFT+T}{SLEEP 300ms}
    cd ~{ENTER}
```

---

## Typical snapshot entry (complete)

```yaml
- app: konsole
  resource_class: org.kde.konsole
  desktop_filename: org.kde.konsole
  desktop: Plumbing
  screen: right
  maximized: true
  minimized: false
  tile: ''
  x: 5760
  y: 1800
  width: 2560
  height: 1440
  command: konsole
  single_instance: false
  ignore_window: false
  tracking_uuid: 3f8a2b1c-4d99-47ce-9a12-e0b87f563d2a
  input_actions: |
    {CTRL+SHIFT+T}{SLEEP 200ms}
    ssh user@server.localhost{ENTER}
  _caption: 'Konsole'
  _pid: 12345
```
