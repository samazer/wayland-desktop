# kwin_script_test.py — Output

**Purpose:** Test whether KWin scripts can be loaded via `/Scripting` and
whether `writeFile()` is available inside the scripting sandbox.

**Key findings:**
- `loadScript()` works with a single path argument (two-argument form rejected by pydbus)
- `start()` is required after `loadScript()`
- `writeFile()` is **not available** — `ReferenceError: writeFile is not defined`
- `print()` inside a KWin script writes to the systemd journal via `kwin_wayland`
- `workspace.windowList()` works and returns 15 windows
- Window properties (internalId, resourceClass, etc.) are all accessible

This result led to ADR-260317-03: use `callDBus()` for communication instead.

---

```
$ python3 kwin_script_test.py
Connected to org.kde.KWin /Scripting
Output file : /tmp/kwin_probe_u7cud9bo.json
Script file : /tmp/kwin_script_18evh0l1.js

KWin script written (2615 bytes)

Loading script via /Scripting ...
loadScript() returned ID: 0
start() called on scripting engine

Waiting for script execution (up to 3s) ...

NOTICE: Output file was NOT written by the script.
        writeFile() is not available in this KWin scripting environment.

Check the KWin journal output for 'kwin_probe:' lines:
  journalctl -b | grep kwin_probe

If RESULTS_JSON appears in the journal, we'll use the journal
as the communication channel instead of a file.

Script ID 0 unloaded cleanly.
```

**Journal output (confirming workspace.windowList() works):**

```
$ journalctl -b | grep kwin_probe
Mar 17 20:58:52 kwin_wayland[2956]: kwin_probe: writeFile not available: ReferenceError: writeFile is not defined
Mar 17 20:58:52 kwin_wayland[2956]: kwin_probe: window count = 15
Mar 17 20:58:52 kwin_wayland[2956]: kwin_probe: RESULTS_JSON={"method":"journal","windows":[
  {"internalId":"{b08b329b-...}","resourceClass":"ibus-ui-gtk3",...},
  {"internalId":"{f2e53258-...}","resourceClass":"org.kde.konsole","caption":"— Konsole",...},
  {"internalId":"{caa3288a-...}","resourceClass":"thunderbird-esr","caption":"Inbox - Mozilla Thunderbird",...},
  ... 15 windows total
]}
```
