# kwin_desktop_assign_test.py — Output

**Purpose:** Test virtual desktop assignment and window focus/raise/minimize
operations — the remaining untested operations before writing the library.

**Key findings:**
- `w.desktops = [desktopObject]` — **works**, window moved to target desktop
- `workspace.activeWindow = w` — **works**, window raised and focused
- `w.minimized = true` — **works**, window minimized
- `w.minimized = false` — **works**, window restored

All confirmed working. See ADR-260317-07 for desktop assignment design.

---

```
$ python3 kwin_desktop_assign_test.py \
    --id "{d6a03aac-8f51-4d99-87ce-48f634523368}"

═══ KWin Desktop Assignment & Focus Test ═══════════════════════
[INFO ] Window ID : {d6a03aac-8f51-4d99-87ce-48f634523368}

Test 1 — Virtual desktop assignment
[INFO ] Available desktops:
[INFO ]    0. Main      4ca398c8-263e-4e5c-a9e8-441353ab0fdf
[INFO ]    1. Job 1     5ea8bd94-1d53-4d22-bf65-9f2044b3edbb
[INFO ]    2. Job 2     bca30267-f55d-42be-9dce-9f80ee8e2c30
[INFO ]    3. Job 3     51f5b20a-dc24-4a3d-bf91-3f3e812f0da7
[INFO ]    4. Plumbing  bc947f74-5e0e-44d0-9731-3e46eee17e5e
[INFO ]    5. Writing   47cf0439-17ed-4fa0-9333-58fe5d8f8383
[INFO ]    6. Task 1    542a8332-6967-4495-b28a-837b1ad89efb
[INFO ]    7. Task 2    477a0ff7-18a9-4e99-a148-26e96aa61da1
[INFO ]    8. Task 3    10fee8f7-9ef6-4e94-af9f-e084e7559ae3
[INFO ]    9. Media     021922a3-8695-4b50-8c00-e034b7c23aa5
[INFO ] Window is on desktop: 'Plumbing'
[INFO ] Target desktop: 'Main'  (4ca398c8-263e-4e5c-a9e8-441353ab0fdf)

[INFO ] Original desktop : 'Plumbing' (bc947f74-5e0e-44d0-9731-3e46eee17e5e)
  ✓  w.desktops = [desktopObject]    WORKS → landed on 'Main'
[OK   ] Restored to original desktop: 'Plumbing'

Test 2 — Focus / raise and minimize / restore

  ✓  workspace.activeWindow = w  (raise/focus)    isActive=True
  ✓  w.minimized = true                           readBack=True
  ✓  w.minimized = false  (restore)               readBack=False

═══ Done ════════════════════════════════════════════════════════

[INFO ] Summary of confirmed working operations for the library:
[INFO ]   • w.desktops = [desktopObj]     (move to virtual desktop)
[INFO ]   • workspace.activeWindow = w    (raise / focus)
[INFO ]   • w.minimized = true/false      (minimize / restore)
[INFO ]   • w.setMaximize(h, v)           (maximize / unmaximize)
[INFO ]   • w.frameGeometry = {x,y,w,h}  (move — x/y only, size ignored by app)
[INFO ]   • Geometry-based screen assign  (move centre-point into screen rect)
```
