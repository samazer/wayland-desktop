# Input Injection Investigation Log

**Date:** 2026-03-22 / 2026-03-23  
**Outcome:** Implemented via XDG RemoteDesktop portal — see ADR-260323-01  
**Test scripts:** `docs/dev/eis_test.py`, `docs/dev/portal_keyboard_test.py`

---

## Summary

KWin's private EIS interface (`org.kde.KWin.EIS.RemoteDesktop`) grants
`POINTER_ABSOLUTE`, `TOUCH`, `BUTTON`, `SCROLL` but withholds `KEYBOARD`.
The XDG RemoteDesktop portal (`org.freedesktop.portal.RemoteDesktop`) grants
all capabilities after a one-time user authorization dialog.

---

## Stage 1 — KWin D-Bus object tree

Confirmed EIS interface location by running `qdbus6 org.kde.KWin`:

```
/org/kde/KWin/EIS
/org/kde/KWin/EIS/InputCapture
/org/kde/KWin/EIS/RemoteDesktop
```

Introspection of `/org/kde/KWin/EIS/RemoteDesktop`:

```
method QDBusUnixFileDescriptor org.kde.KWin.EIS.RemoteDesktop.connectToEIS(int capabilities, int& cookie)
method void org.kde.KWin.EIS.RemoteDesktop.disconnect(int cookie)
```

Return type is `QDBusUnixFileDescriptor` — the fd index in a Unix fd list,
not a raw integer.  pydbus returns the index (0); must use
`Gio.DBusConnection.call_with_unix_fd_list_sync()` to extract the real fd.

---

## Stage 2 — eis_test.py: initial run (pydbus — wrong fd)

```
$ python3 docs/dev/eis_test.py
=== EIS Diagnostic ===

[OK] Loaded libei.so.1
[OK] ei_event_get_type available

--- Step 1: KWin EIS D-Bus connection ---
Requesting capabilities: 54 (KEYBOARD|POINTER_ABSOLUTE|BUTTON|SCROLL)
[OK] connectToEIS returned fd=0, cookie=3

--- Step 2: libei context ---
[OK] ei_new_sender -> 94352175424256
[OK] ei_setup_backend_fd -> 0
[OK] ei_get_fd -> 7

--- Step 3: Event loop (5 seconds) ---

--- Step 4: Test keystroke ---
[SKIP] No keyboard device — cannot send keystroke

--- Summary ---
Events received:  0
```

**Finding:** `fd=0` is stdin.  pydbus returns the fd list index, not the fd.

---

## Stage 3 — eis_test.py: after Gio fd fix

After switching to `Gio.DBusConnection.call_with_unix_fd_list_sync()`:

```
[OK] connectToEIS: fd_index=0, real_fd=7, cookie=4
```

Real fd is now 7.  But `ei_seat_bind_capabilities` caused a segfault:

```
  EVENT #3: type=3  seat=94191931172800  device=NULL
    -> Binding capabilities on seat 94191931172800
Segmentation fault         (core dumped)
```

**Finding:** `ei_seat_bind_capabilities` is variadic.  Without explicit
`argtypes`, ctypes truncated the 64-bit seat pointer.  Declaring full
argtypes for the fixed 4-capability call fixed the segfault.

---

## Stage 4 — eis_test.py: after argtypes fix — seat confirmed, no keyboard

```
$ python3 docs/dev/eis_test.py
=== EIS Diagnostic ===

[OK] Loaded libei.so.1
[OK] ei_event_get_type available

--- Step 1: KWin EIS D-Bus connection ---
Requesting capabilities: 54 (KEYBOARD|POINTER_ABSOLUTE|BUTTON|SCROLL)
[OK] connectToEIS: fd_index=0, real_fd=7, cookie=5

--- Step 2: libei context ---
[OK] ei_new_sender -> 93846608493600
[OK] ei_setup_backend_fd -> 0
[OK] ei_get_fd -> 9

--- Step 3: Event loop (5 seconds) ---
  EVENT #1: type=91  seat=NULL  device=NULL
  EVENT #2: type=1  seat=NULL  device=NULL
  EVENT #3: type=3  seat=93846606662208  device=NULL
    -> Binding capabilities on seat 93846606662208
  EVENT #4: type=5  seat=93846606662208  device=94657296930784
    -> Device capabilities: ['POINTER_ABSOLUTE', 'TOUCH', 'SCROLL', 'BUTTON']
    -> POINTER_ABSOLUTE device stored
  EVENT #5: type=8  seat=93846606662208  device=94657296930784
    -> Device capabilities: ['POINTER_ABSOLUTE', 'TOUCH', 'SCROLL', 'BUTTON']
    -> POINTER_ABSOLUTE device stored

--- Step 4: Keyboard-only EIS connection ---
(Testing whether KWin grants keyboard when requested alone)
[OK] connectToEIS(KEYBOARD only): fd=10, cookie=8
  KBD-ONLY EVENT type=91  seat=NULL  device_caps=NULL
  KBD-ONLY EVENT type=1   seat=NULL  device_caps=NULL
  KBD-ONLY EVENT type=3   seat=yes (caps=['TOUCH'])  device_caps=NULL

--- Summary ---
Events received:  5
Seat bound:       True
Keyboard device:  NOT READY
Pointer device:   READY
```

**Key finding:** `seat caps=['TOUCH']` — KWin's private EIS interface only
advertises TOUCH capability.  KEYBOARD is deliberately withheld.  No
amount of `ei_seat_bind_capabilities(KEYBOARD)` calls will produce a
keyboard device.  The private interface cannot be used for keyboard injection.

---

## Stage 5 — portal_keyboard_test.py: first run

Path construction bug caused immediate failure (response=1):

```
--- Step 1: CreateSession ---
[FAIL] CreateSession response=1  results={}
```

**Finding:** Request object path must be
`/org/freedesktop/portal/desktop/request/{sender}/{token}` where sender
is `:1.183` → `1_183`.  The helper was prepending `kwin_desk_`, making
paths that never matched the portal's emitted signal paths.

Also: all three setup calls (CreateSession, SelectDevices, Start) are
async — they return immediately and fire a `Response` signal on the
request path.  CreateSession was incorrectly treated as synchronous.

Also: Gio `signal_subscribe` callbacks take **7** arguments, not 6 —
the last is `user_data` passed at subscription time.

---

## Stage 6 — portal_keyboard_test.py: successful run

After fixing all three issues:

```
$ python3 docs/dev/portal_keyboard_test.py
=== Portal Keyboard Test ===

[INFO] No saved token — dialog will appear

--- Step 1: CreateSession ---
[OK] CreateSession response=0  results={'session_handle': '/org/freedesktop/portal/desktop/session/1_189/sikkpnkt'}
[OK] session_path: /org/freedesktop/portal/desktop/session/1_189/sikkpnkt

--- Step 2: SelectDevices ---
[OK] SelectDevices response=0  results={}

--- Step 3: Start ---
(If the KDE dialog appears, click 'Allow')
[OK] Start response=0
[OK] Restore token saved: 47ec9d5d-6f8b-4c...

--- Step 4: Send keystroke ---
Sending 't' (keycode 20) to the focused window in 2 seconds...
Focus a terminal or text editor now!
[OK] Keystroke sent — did a 't' appear?

--- Step 5: Close session ---
[OK] Session closed

=== Done ===
```

KDE authorization dialog appeared; user approved.  Restore token saved.
Keystroke `t` was delivered to the focused terminal.  **Full stack confirmed working.**

---

## Stage 7 — kwin_restore.py phase 4: first attempt (GLib Variant bug)

After integrating into `input_injection.py`, all actions failed with error code 0:

```
Phase 4 -- Input actions
[INFO ]   Input actions: 'google-chrome'
[WARN ]     Action 'key' failed (non-fatal): 0
[WARN ]     Action 'text' failed (non-fatal): 0
...
[OK   ]     Done: 0 action(s), 58 skipped
```

**Finding:** Pre-constructing `GLib.Variant("a{sv}", {})` and passing it
inside another `GLib.Variant("(oa{sv}iu)", (...))` wraps it as type `v`
(generic variant) not `a{sv}` (inline dict).  The portal rejects the
wrong signature silently with error code 0.

Fix: pass `{}` directly as a Python dict.  PyGObject infers `a{sv}` from
the outer format string correctly.

---

## Stage 8 — kwin_restore.py phase 4: success

```
Phase 4 -- Input actions
[INFO ]   Input actions: 'google-chrome'
  [DBG] get_screen_geometry  -> {'ok': True, 'win_x': 3845, ...}  (1ms)
  [DBG] raise_window         -> {'ok': True, 'op': 'raise_window', 'elapsed_ms': 200}  (200ms)
[OK   ]     Done: 58 action(s)
[INFO ]   Input actions: 'google-chrome'
  [DBG] get_screen_geometry  -> {'ok': True, 'win_x': 2885, ...}  (0ms)
  [DBG] raise_window         -> {'ok': True, 'op': 'raise_window', 'elapsed_ms': 200}  (200ms)
[OK   ]     Done: 58 action(s)
[INFO ]   Input actions: 'google-chrome'
  [DBG] get_screen_geometry  -> {'ok': True, 'win_x': 0, 'win_y': 1080, ...}  (0ms)
  [DBG] raise_window         -> {'ok': True, 'op': 'raise_window', 'elapsed_ms': 200}  (200ms)
[OK   ]     Done: 121 action(s)

Summary
[OK   ] Placed   : 3

=== Done ============================================================
```

All 58/58/121 actions succeeded.  URL was injected into Chrome's address
bar via `{CTRL+L}https://.../{ENTER}` and the page loaded.

---

## Chronological issue list

| # | Issue | Root cause | Fix |
|---|---|---|---|
| 1 | `fd=0` (stdin) returned | pydbus returns fd list index, not fd | Use `Gio.call_with_unix_fd_list_sync()` |
| 2 | Zero events received | fd=0 meant EIS connected to stdin | Fixed by #1 |
| 3 | Segfault in `ei_seat_bind_capabilities` | Variadic C function; no argtypes; pointer truncated | Declare explicit argtypes for fixed 4-cap call |
| 4 | No keyboard device ever arrives | KWin private EIS withholds KEYBOARD | Use XDG RemoteDesktop portal instead |
| 5 | `CreateSession response=1` immediately | Request path had `kwin_desk_` prefix in sender segment | `sender = name.lstrip(":").replace(".", "_")` only |
| 6 | Same path bug in SelectDevices/Start | Tokens also had `kwin_desk_` prefix | `_rand_token()` returns plain 8-char string |
| 7 | CreateSession treated as synchronous | All three setup calls are async Request/Response | Subscribe to Response signal and run GLib loop for each |
| 8 | Callback TypeError (6 args, 7 given) | Gio signal callbacks always receive user_data as 7th arg | Add `user_data` to all three response callbacks |
| 9 | All Notify* calls fail with error code 0 | Pre-built `GLib.Variant("a{sv}", {})` passed as `v` not `a{sv}` | Pass `{}` dict directly in outer Variant constructor |
