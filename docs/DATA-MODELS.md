<!-- SPDX-FileCopyrightText: 2026 Sam Azer <sam dot azer at azertech dot net>  -->
<!-- SPDX-License-Identifier: CC-BY-SA-4.0  -->
# wayland-desktop — Data Models

The three core dataclasses used throughout the library.  All are defined in
`kwin_lib/models.py` and imported from `kwin_lib` directly.
---

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
- [WindowInfo](#windowinfo)
- [DesktopInfo](#desktopinfo)
- [ScreenInfo](#screeninfo)

---

### Copyright &amp; License
Wayland Desktop Restore (documentation) Copyright © 2026 by Sam Azer &lt;sam dot azer at azertech dot net&gt; is licensed under Creative Commons Attribution-ShareAlike 4.0 International. To view a copy of this license, visit https://creativecommons.org/licenses/by-sa/4.0/

#### Creative Commons Attribution-ShareAlike 4.0 International
This license requires that re-users give credit to the creator. It allows re-users to distribute, remix, adapt, and build upon the material in any medium or format, even for commercial purposes. If others remix, adapt, or build upon the material, they must license the modified material under identical terms.

- Credit must be given to the creator.
- Adaptations must be shared under the same terms.

[See the full License](https://creativecommons.org/licenses/by-sa/4.0/)

---

## WindowInfo

Returned by `WindowProbe.run()`, `WindowManager.get_info()`, and
`WindowManager.find_by_class()`.  The `internalId` field is the handle
passed to all `WindowManager` operations.

```python
@dataclass
class WindowInfo:
    internalId:      str    # UUID — use this as the handle for operations
    detected_ms:     int    # ms after launch when window appeared; 0 = pre-existing
    resourceClass:   str    # KWin's class identifier, e.g. "thunderbird-esr"
    resourceName:    str
    caption:         str    # window title
    desktopFileName: str    # base name of the app's .desktop file
    pid:             int
    desktop_id:      str    # UUID of current virtual desktop
    desktop_name:    str    # name of current virtual desktop
    screen_name:     str    # connector name of current screen
    x:               int
    y:               int
    width:           int
    height:          int
    minimized:       bool
    maximized:       bool
    fullScreen:      bool
    onAllDesktops:   bool

    # Computed
    is_pre_existing: bool   # True when detected_ms == 0
```

---

## DesktopInfo

Returned by `DesktopManager.list_desktops()` and `DesktopManager.find_desktop()`.
The `id` field (a UUID string) is required when calling `WindowManager.move_to_desktop()`.

```python
@dataclass
class DesktopInfo:
    id:     str    # UUID — required for desktop assignment operations
    name:   str    # "Main", "Job 1", etc.
    number: int    # 1-based
```

---

## ScreenInfo

Returned by `DesktopManager.list_screens()` and `DesktopManager.find_screen()`.
The `name` field is the KWin connector name (e.g. `"HDMI-A-2"`) used by
`WindowManager.move_to_screen()`.

```python
@dataclass
class ScreenInfo:
    name:         str    # connector name, e.g. "HDMI-A-2"
    x:            int    # top-left in compositor coordinates
    y:            int
    width:        int
    height:       int
    model:        str
    manufacturer: str

    # Computed
    centre_x:     int
    centre_y:     int
```
