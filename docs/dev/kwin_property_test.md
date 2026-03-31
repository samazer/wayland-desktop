# kwin_property_test.py — Output

**Purpose:** Test whether KWin's JS scripting sandbox exposes Qt's dynamic
property mechanism (`setProperty` / `property`) on window objects.

**Status: NOT YET RUN** — this is the first task for the next development
session.  See PROBLEMS.md open issue 4.

---

## How to run

```bash
# Get a window UUID first
python3 kwin_probe.py -- thunderbird

# Then run the property test
python3 docs/dev/kwin_property_test.py --id "{uuid-from-above}"
```

## What to look for

The test has three checkpoints:

| Test | Question | Impact if fails |
|---|---|---|
| Test 1 | Is `setProperty`/`property` callable at all? | Entire UUID approach is dead |
| Test 2 | Does the property persist across separate script loads? | UUID approach is dead |
| Test 3 | Can we find a window by UUID scan? | Lookup pattern for realign utility |

## Expected output (if all tests pass)

```
Test 1 — setProperty() write + immediate read (same script)
  ✓  setProperty is a function                    True
  ✓  property is a function                       True
  ✓  setProperty('kwinlib_uuid', value) — write   returned 'true'
  ✓  property('kwinlib_uuid') — read back         'test-kwinlib-a3f2b8c1-4d99-87ce'
  ✓  Round-trip value matches written value       True

Test 2 — Property persistence across separate script loads
  OK  Script A wrote property: 'test-kwinlib-a3f2b8c1-4d99-87ce-cross'
  ✓  Property readable from second script load    'test-kwinlib-...-cross'
  ✓  Value matches what was written               True
  CONFIRMED: Qt dynamic properties persist on window QObjects
  The session UUID tracking approach is viable.

Test 3 — Find window by kwinlib_uuid property (lookup pattern)
  ✓  Window found by UUID scan (searched N windows)
  ✓  internalId matches original target           True
```

## If tests fail

Record the actual output here and update PROBLEMS.md issue 4 with the
failure mode and chosen alternative approach.
