# kwin_mutation_test.py — Output

**Purpose:** Verify the KWin scripting API for window mutation operations and
screen/output enumeration before writing the window management library.

**Key findings:**
- `workspace.desktops` — works, returns full UUID + name + number for all desktops
- `workspace.screens` — works, returns all 5 screens with names and geometry
- `internalId` lookup via `workspace.windowList()` — reliable
- `w.desktops[0].id` / `w.output.name` — current placement readable
- `w.setMaximize(true, true)` — works
- `w.frameGeometry` — readable
- `w.output` — **read-only** (TypeError on assignment attempt)
- `sendToOutput()` — **does not exist** on window objects
- `Qt.rect()` — **not available** (`ReferenceError: Qt is not defined`)

See ADR-260317-06 (screen assignment) and ADR-260317-07 (desktop assignment).

---

## Pass 1 — Read-only, no window ID

```
$ python3 kwin_mutation_test.py

═══ KWin Mutation & Screen API Test ═══════════════════════════
[INFO ] Target window : (none — Tests C/D/E will be skipped)
[INFO ] Mutate        : False
[OK   ] Infrastructure ready — running tests

Test A — Virtual desktop enumeration
[INFO ] workspace.desktops result:
  source    'workspace.desktops'
  desktops:
    id      '4ca398c8-263e-4e5c-a9e8-441353ab0fdf'  name 'Main'      number 1
    id      '5ea8bd94-1d53-4d22-bf65-9f2044b3edbb'  name 'Job 1'     number 2
    id      'bca30267-f55d-42be-9dce-9f80ee8e2c30'  name 'Job 2'     number 3
    id      '51f5b20a-dc24-4a3d-bf91-3f3e812f0da7'  name 'Job 3'     number 4
    id      'bc947f74-5e0e-44d0-9731-3e46eee17e5e'  name 'Plumbing'  number 5
    id      '47cf0439-17ed-4fa0-9333-58fe5d8f8383'  name 'Writing'   number 6
    id      '542a8332-6967-4495-b28a-837b1ad89efb'  name 'Task 1'    number 7
    id      '477a0ff7-18a9-4e99-a148-26e96aa61da1'  name 'Task 2'    number 8
    id      '10fee8f7-9ef6-4e94-af9f-e084e7559ae3'  name 'Task 3'    number 9
    id      '021922a3-8695-4b50-8c00-e034b7c23aa5'  name 'Media'     number 10
  count   10

Test B — Screen / output enumeration
[INFO ] Screen API     : workspace.screens
[INFO ] Screen count   : 5
[INFO ] Active screen  : HDMI-A-2

  Screen 0: 'DP-4'      geometry: 2885,0    1920x1080
  Screen 1: 'HDMI-A-2'  geometry: 1920,1080 3840x2160
  Screen 2: 'DP-5'      geometry: 5760,1800 2560x1440
  Screen 3: 'Unknown-2' geometry: 0,2160    1920x1080
  Screen 4: 'HDMI-A-1'  geometry: 0,1080    1920x1080
```

## Pass 2 — With Thunderbird window ID (Tests C and D)

```
$ python3 kwin_mutation_test.py --id "{d6a03aac-8f51-4d99-87ce-48f634523368}"

Test C — Find window by internalId
[OK   ] Window found (searched 15 windows)
[INFO ]   resourceClass : 'thunderbird-esr'
[INFO ]   caption       : 'Inbox - sam@azer.ca (mail6) - Mozilla Thunderbird'
[INFO ]   geometry      : 1920,1080  3840x2110
[INFO ]   desktop id    : 'bc947f74-5e0e-44d0-9731-3e46eee17e5e'
[INFO ]   minimized     : False
[INFO ]   maximized     : False

Test D — Window desktop/screen property inspection
[OK   ] Window placement properties:
  onAllDesktops        False
  desktopsType         'object'
  desktopsLen          1
  desktop0Id           'bc947f74-5e0e-44d0-9731-3e46eee17e5e'
  desktop0Name         'Plumbing'
  desktop0Number       5
  outputName           'HDMI-A-2'
  outputType           'object'
  hasSendToOutput      False
  hasSetMaximize       True
  frameGeometry        {'x': 1920, 'y': 1080, 'width': 3840, 'height': 2110}
```

## Pass 3 — With --mutate flag

```
$ python3 kwin_mutation_test.py --id "{d6a03aac-...}" --mutate

Test E — Mutation tests (--mutate)
[WARN ] These tests WILL briefly move/resize the target window.

[WARN ] Partial success (3 error(s)):
  ✓ maximized
  ✗ resize: ReferenceError: Qt is not defined
  ✗ move: ReferenceError: Qt is not defined
  ✗ restore: ReferenceError: Qt is not defined
[INFO ] Original geometry restored: 1920,1080  3840x2110
```
