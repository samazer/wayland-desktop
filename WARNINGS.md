<!-- SPDX-FileCopyrightText: 2026 Sam Azer <sam dot azer at azertech dot net>  -->
<!-- SPDX-License-Identifier: GPL-3.0-or-later  -->
# WARNINGS

These notes are provided as a convenience.  This document contains information about issues that are known at the time of writing.  You may find additional issues to consider if you decide to try this software.

---

## Available Documents
| Document                                     | Description                                                                    |
|:---------------------------------------------|:-------------------------------------------------------------------------------|
| [docs/OVERVIEW.md](docs/OVERVIEW.md)         | How the library works                                                          |
| [docs/INSTALL.md](docs/INSTALL.md)           | Insetallation and setup information.                                           |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | System context and design overview                                             |
| [docs/PROBLEMS.md](docs/PROBLEMS.md)         | Known limitations and open issues                                              |
| [docs/GLOSSARY.md](docs/GLOSSARY.md)         | Definitions of terms used in the codebase and documentation                    |
| [docs/ADRs.md](docs/ADRs.md)                 | Architectural Decision Records                                                 |
| [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md) | Project requirements and implementation status                                 | [docs/DATA-MODELS.md](docs/DATA-MODELS.md) | WindowInfo, DesktopInfo, ScreenInfo dataclasses |
| [docs/LIBRARY-API.md](docs/LIBRARY-API.md)   | Python API reference (KWinSession, WindowProbe, WindowManager, DesktopManager) |
| [docs/CONFIG.md](docs/CONFIG.md)             | Configuration file for all wayland-desktop scripts                             |
| [docs/SNAPSHOT.md](docs/SNAPSHOT.md)         | Documentation for the Desktop Setup (YAML) file                                |
| [README](README.md)                          | The introductory text and User Documentation for this project                  |
| [ABOUT.md](ABOUT.md)         | Some background information about this project |


--

## Contents

0. [Copyright &amp; License](#copyright--license)
1. [Desktop Input Injection &amp; Window Focus](#copyright--license-cl)

## Copyright &amp; License
<!-- do not edit: begin -->
**Wayland Desktop Setup**<br>
*Plasma6 Session Restore for Wayland, including input injection.*

Copyright © 2026 by Sam Azer&lt;sam dot azer at azertech dot net&gt;, All rights reserved.

This software is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or (at
your option) any later version.

This software is distributed in the hope that it will be useful, but
WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the [GNU
General Public License](LICENSE.md) for more details.

You should have received a copy of the [GNU General Public
License](LICENSE.md) along with this program.  If not, see
<https://www.gnu.org/licenses/>.

<!-- do not edit: end -->

# Desktop Input Injection &amp; Window Focus

The Input Injection feature, which is used by assigning simple scripts to the `input_actions` properties of the `setup.yaml` file ([see the documentation](docs/SNAPSHOT.md),) works by sending a stream of tokens into the *desktop*.

- For this to work properly, *the script must successfully focus the correct target window.*
- Additionally, the script must re-focus the window in cases where:
   - the system is expected to shift the focus to another window, such as when an open-file dialog is triggered,
   - and after any lengthy delays, during which the focus might change.
   - See the [SNAPSHOTS](docs/SNAPSHOT.md) document for details on the scripting features associated with the `input_actions` property of the setup.yaml files.  In particular, understand the [Control Tokens](docs/SNAPSHOT.md#control-tokens) such as `{FOCUS-MAIN}`, `{FOCUS-TOOL}`, `{ACTIVE-WINDOW}` and `{DELAY <n>}`.

`input_actions` need to be thought out carefully to avoid problems.  Test your Session Restore setup carefully and pay close attention to everything that happens.  Try to guess where something might not work in the event of a problem.  Close all your windows and run the restore again (and again!) until you get everything working reliably.

This may be a complicated and time-consuming task, so, if you have doubts, disable the Secret Service and delete all the `input_actions` properties from your setup files.

When the provided features are used properly, making sure to delay input injection when the system might be busy and focus the correct target window before sending keystrokes, the input injection feature should be useful and effective.

