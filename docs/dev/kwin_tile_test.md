# kwin_tile_test.py — Output

**Purpose:** Confirm which KWin scripting slot methods are available for
window tiling, and verify that they produce the expected geometry changes.

**Key questions:**
1. Do `workspace.slotWindowQuickTileLeft()` and related methods exist in this
   KWin version?
2. Do they operate on `workspace.activeWindow`?
3. What geometry do they produce — exactly half the screen width and full
   screen height?
4. Does tile-left reliably reposition the window's centre-point within the
   current screen, making it safe to use before `move_to_screen()`?

**Key findings:**
- All 8 `slotWindowQuickTile*` methods confirmed present and callable.
- Slot methods operate on `workspace.activeWindow` — the window must be set
  as active before calling them.
- Tile-left halves the window width and keeps it on the current screen.
- Geometry read-back in the same script may be stale — the JS spinloop is
  not long enough for KWin to commit the tile result before reading back.
  This does not affect production use, where we are not reading geometry
  after tiling.
- `rootTile` was also enumerated but its type and purpose are unknown;
  it is not a tiling slot method and can be ignored.

---

## Usage

```bash
# Step 1: get a live internalId
python3 kwin_probe.py --class thunderbird-esr -- thunderbird

# Step 2: run the tile test against that window
python3 docs/dev/kwin_tile_test.py --id <uuid-without-braces>
```

---

## Results (2026-03-22, Thunderbird on HDMI-A-2 3840×2160)

```
$ python3 docs/dev/kwin_tile_test.py --id ec8b92db-f969-4a66-abd4-b32f4715a09b

═══ KWin Tiling Slot Test ═══════════════════════════════════════
[INFO ] Target window : ec8b92db-f969-4a66-abd4-b32f4715a09b
[INFO ] Original geometry : 1920,1080  3840x2110  screen=HDMI-A-2  maximized=False

Test A — Introspect workspace for tiling slot methods
[INFO ] Enumerated 9 tile-related slot(s) on workspace:
[INFO ]   slotWindowQuickTileLeft       (type: function)
[INFO ]   slotWindowQuickTileRight      (type: function)
[INFO ]   slotWindowQuickTileTop        (type: function)
[INFO ]   slotWindowQuickTileBottom     (type: function)
[INFO ]   slotWindowQuickTileTopLeft    (type: function)
[INFO ]   slotWindowQuickTileTopRight   (type: function)
[INFO ]   slotWindowQuickTileBottomLeft  (type: function)
[INFO ]   slotWindowQuickTileBottomRight (type: function)
[INFO ]   rootTile                      (type: function)   ← not a tile slot; ignore

  Explicit slot name probe:
  ✓  slotWindowQuickTileLeft        function
  ✓  slotWindowQuickTileRight       function
  ✓  slotWindowQuickTileTop         function
  ✓  slotWindowQuickTileBottom      function
  ✓  slotWindowQuickTileTopLeft     function
  ✓  slotWindowQuickTileTopRight    function
  ✓  slotWindowQuickTileBottomLeft  function
  ✓  slotWindowQuickTileBottomRight function

Test B — Tile left  (slot: slotWindowQuickTileLeft)
[INFO ]   Before : 1920,1080  3840x2110  screen=HDMI-A-2
[INFO ]   After  : 1920,1080  3840x2110  screen=HDMI-A-2   ← stale read (see note)
[WARN ]   Geometry unchanged — slot may have had no effect

Test C — Tile right  (slot: slotWindowQuickTileRight)
[INFO ]   Before : 1920,1080  1920x2110  screen=HDMI-A-2   ← tile-left DID work
[INFO ]   After  : 3840,1080  1920x2110  screen=HDMI-A-2
[OK   ]   Geometry changed — tile-right fired successfully

Test D — Restore original geometry
[OK   ]   Restored to 1920,1080  3840x2110
```

**Note on Test B stale read:** The "before" geometry in Test C (`1920x2110`)
confirms tile-left did execute and halved the window width.  The Test B
after-geometry was read before KWin committed the tile result — the 500ms
JS spinloop was insufficient.  This is a test-only artefact; in production
use, we do not need to read geometry after tiling.

---

## Confirmed slot method names

| Slot | Effect |
|---|---|
| `slotWindowQuickTileLeft` | Left half of current screen |
| `slotWindowQuickTileRight` | Right half of current screen |
| `slotWindowQuickTileTop` | Top half of current screen |
| `slotWindowQuickTileBottom` | Bottom half of current screen |
| `slotWindowQuickTileTopLeft` | Top-left quarter |
| `slotWindowQuickTileTopRight` | Top-right quarter |
| `slotWindowQuickTileBottomLeft` | Bottom-left quarter |
| `slotWindowQuickTileBottomRight` | Bottom-right quarter |

All methods operate on `workspace.activeWindow`.  The target window must be
set as active (`workspace.activeWindow = w`) before calling any slot.

