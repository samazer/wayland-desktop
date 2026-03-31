# kwin_dbus_callback_test.py — Output

**Purpose:** Test whether a KWin JS script can call back to a Python-registered
D-Bus service using `callDBus()`, and whether `workspace.windowList()` data
can be returned via that channel.

**Key findings:**
- `callDBus()` **works** — confirmed by ping round-trip
- Full window list (15 windows, 4483 bytes of JSON) transmitted successfully
- `internalId` UUIDs are already brace-wrapped: `{b08b329b-...}`
- All expected properties present: resourceClass, pid, geometry, desktops, output
- This result established the core communication architecture for the library

See ADR-260317-03.

---

```
$ python3 kwin_dbus_callback_test.py
Registering D-Bus receiver service ...
  Published as 'org.probe.KWinReceiver' at '/Receiver'
  GLib main loop running in background thread
  Connected to org.kde.KWin /Scripting

KWin script written to: /tmp/kwin_cbtest_dlw6wk3c.js
Loading KWin script ...
  Script loaded (ID: 2), start() called

Waiting for ping callback (up to 3s) ...
  [D-Bus] ping() received: 'hello from KWin script'
  SUCCESS: ping round-trip confirmed — callDBus() works!
Waiting for snapshot callback (up to 3s) ...
  [D-Bus] receiveSnapshot() received (4483 bytes)
  SUCCESS: received 15 windows via D-Bus callback

  [0]  ibus-ui-gtk3                    pid=2956
        caption='ibus-ui-gtk3'
        internalId={b08b329b-7150-4ebb-865f-342ac1964320}
  [1]  plasmashell                     pid=3314
        caption=''
        internalId={c82b2f4e-05e3-4b9b-be49-a40a17cd6f92}
  [2]  plasmashell                     pid=3314
        caption=''
        internalId={642f71af-54ff-4e5f-aa98-51490ef2b4fc}
  [3]  plasmashell                     pid=3314
        caption=''
        internalId={c9f56ee4-30e9-4b7d-8cc0-8b61fd6adf1e}
  [4]  plasmashell                     pid=3314
        caption=''
        internalId={51aa611b-73b0-467a-8b67-59e99d55a630}
  [5]  plasmashell                     pid=3314
        caption=''
        internalId={eb6f263a-8d43-429c-b4b6-57749fbdd1e4}
  ... and 9 more

Script ID 2 unloaded.
D-Bus service unpublished.

Done.
```
