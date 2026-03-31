<!-- SPDX-FileCopyrightText: 2026 Sam Azer <sam dot azer at azertech dot net>  -->
<!-- SPDX-License-Identifier: GPL-3.0-or-later  -->
# About this project

Some background information about this project.

---

## Available Documents
| Document                                     | Description                                                                                   |
|:---------------------------------------------|:----------------------------------------------------------------------------------------------|
| [docs/OVERVIEW.md](docs/OVERVIEW.md)         | How the library works                                                                         |
| [docs/INSTALL.md](docs/INSTALL.md)           | Insetallation and setup information.                                                          |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | System context and design overview                                                            |
| [docs/PROBLEMS.md](docs/PROBLEMS.md)         | Known limitations and open issues                                                             |
| [WARNINGS.md](WARNINGS.md)                   | User-level Usage Warnings                                                                     |
| [docs/GLOSSARY.md](docs/GLOSSARY.md)         | Definitions of terms used in the codebase and documentation                                   |
| [docs/ADRs.md](docs/ADRs.md)                 | Architectural Decision Records                                                                |
| [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md) | Project requirements and implementation status                                                | [docs/DATA-MODELS.md](docs/DATA-MODELS.md) | WindowInfo, DesktopInfo, ScreenInfo dataclasses |
| [docs/LIBRARY-API.md](docs/LIBRARY-API.md)   | Python API reference (KWinSession, WindowProbe, WindowManager, DesktopManager)                |
| [docs/CONFIG.md](docs/CONFIG.md)             | Configuration file for all wayland-desktop scripts                                            |
| [docs/SNAPSHOT.md](docs/SNAPSHOT.md)         | Documentation for the Desktop Setup (YAML) file                                               |
| [ABOUT.md](ABOUT.md)         | Some background information about this project |

---

## Contents

I started studying AI a few years ago when a friend produced a Raspberry Pi box that could recognize cats. It was 
interesting.  Of course there were no big LLM's in those days (or none that we knew about.)  We didn't know about 
Huggingface back then, for example.

Recently, I got stuck on a small problem and was forced to switch from X11 to Wayland.  As you know, the Wayland Session 
Restore is not working yet and is planned for late this year (2026.)  The lack of support for Session Restore in Wayland 
exaggerated the work needed to get started after a reboot.  So, I started asking questions on claude.ai.

This led to *Wayland Desktop Setup* being my first serious effort to work with Claude.ai to produce a software tool.

Long story short... Claude was able to search documentation much faster than any human, sometimes doing hours worth of 
research in minutes.  Also, Claude wrote test cases for various issues, including testing and verifying assumptions.  
Much of that work has been preserved in the `docs/dev/` folder.  At some point I was rather impressed with what I was 
seeing.  I started asking Claude to maintain a growing list of documents (see the docs folder,) as well as building 
various tools and libraries.

Again, I was surprised that the code was running and working (after at least two years of systematically deleting almost 
anything that was written by an A.I.)  In addition, I was mostly able to tell when Claude was going down a bad 
rabbit-hole or when Claude was suggesting something that made no sense.  For example, at one point I asked for a simple 
integer counter and Claude wanted to create an array structure for holding the count... Crazy.  

It's strange, really.  The interaction with the A.I is not like an adult working with a child.  It's also not like a guy 
working with an idiot-savant.  It's strange.

Anyway, the AI has become a useful tool and it has helped me produce a Desktop Session Setup for Wayland that is better 
than the original X11 version.

Be aware that the input injection feature is not for the faint of heart.  You really need to test your input injection 
scripts carefully as there are a few ways that your system may focus the wrong window (delays that need to be longer, 
failure to re-focus a window that has lost focus for some reason, etc., keep testing until you get it right.) If you 
are not careful, you can find yourself sending Secret Service tokens to the wrong window, including to browser pages
that might expose your secrets.  

Also be aware that the feature which handles input injection of Mouse operations is not working at all just now.  The 
mouse clicks find their way to the wrong parts of the screen/window.  

Aside from that, wayland-desktop allows me to reboot into a Wayland session and get started much faster than was 
possible on X11.

Hopefully there will be other people who will benefit from this project.  Let me know.

Thanks,<br>
All the best,<br>
Sam &lt;sam dot azer at azertech dot net&gt;


