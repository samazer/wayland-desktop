# kwin_geometry_screen_test.py — Output

**Purpose:** Targeted fix test for two issues found in `kwin_mutation_test.py`:
`Qt.rect()` unavailable for geometry setting, and `w.output` being read-only
for screen assignment.

**Key findings:**
- All three geometry setter approaches **silently fail** for size changes —
  position changes work, but Wayland apps ignore compositor-initiated resize
- `w.output = screenObject` — **TypeError: Cannot assign to read-only property**
- Geometry-based screen assignment (move centre-point into target screen rect)
  **works** — confirmed by reading back `w.output.name` after the move

See ADR-260317-06.

---

```
$ python3 kwin_geometry_screen_test.py \
    --id "{d6a03aac-8f51-4d99-87ce-48f634523368}"

═══ KWin Geometry & Screen Assignment Test ═════════════════════
[INFO ] Window ID     : {d6a03aac-8f51-4d99-87ce-48f634523368}
[INFO ] Target screen : (will pick automatically)

Test 1 — Geometry setter alternatives (no Qt.rect)
[INFO ] Original geometry: 1920,1080  1920x1120

  ✗  Plain JS object  { x, y, width, height }  SILENT FAIL — read back 1920x1120
  ✗  w.geometry = { x, y, width, height }       SILENT FAIL — read back 1920x1120
  ✗  Object.create(null) with properties        SILENT FAIL — read back 1920x1120

[OK   ] Original geometry restored: 1920,1080  1920x1120
[WARN ] No geometry setter approach worked — library will need workaround
[INFO ] Window is on   : 'HDMI-A-2'
[INFO ] Auto-selected  : 'DP-4'

Test 2 — Screen assignment  (target: 'DP-4')
[INFO ] Original screen : 'HDMI-A-2'
[INFO ] Target screen   : 'DP-4'  (2885,0  1920x1080)

  ✗  Direct: w.output = screenObject
        ERROR: TypeError: Cannot assign to read-only property "output"
  ✓  Geometry: move centre-point into screen rect
        WORKS  (read back 'DP-4')
[INFO ]   directAssign after 300ms pause: 'HDMI-A-2'

[OK   ] Original position restored

[OK   ] Best approach for library: 'geometryBased'

═══ Done ════════════════════════════════════════════════════════
```

**Note on geometry setter "silent fail":** Position changes (x, y) do work via
`frameGeometry` assignment. Only size (width, height) is silently ignored — this
is expected Wayland behaviour. The screen test confirmed position-setting works
by successfully moving the window to a different screen.
