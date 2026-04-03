---
phase: 09-update-panels-to-use-new-ct-meter-data-for-grid-energy-import-current-voltage-real-power-instead-of-smart-meter
plan: "01"
subsystem: dashboard
tags: [grafana, influxdb, sql, ct-meter, smart-meter, grid]

requires:
  - phase: 08-backfeed-detection-and-logging
    provides: Backfeed panels 33/34/35 that now point to CT grid table

provides:
  - 12 dashboard panels migrated from Smart Meter to CT grid meter table
  - Signed power convention (negative = backfeed) in Panel 801 Grid series
  - Accurate real-time grid voltage, current, and power from CT sensor

affects:
  - Any future phase modifying grid-related panels

tech-stack:
  added: []
  patterns:
    - "CT grid table uses clean field names (power, voltage, current) without _sensor suffix"
    - "Panel 801 Grid series: no GREATEST() clip — allows negative values for backfeed"
    - "Mixed-source panels (20, 21): CT for electrical measurements, Smart Meter for derived values (frequency, PF)"

key-files:
  created: []
  modified:
    - solar-pv-monitor.json

key-decisions:
  - "Panels 17 (Frequency) and 18 (Power Factor) intentionally stay on Smart Meter — CT table has no frequency or power_factor fields"
  - "Panel 801 Grid series: removed GREATEST() clip to allow negative kW during backfeed (new behavior, correct)"
  - "Panel 20 refId=A (Voltage) migrated to CT; refId=B (Frequency) stays on Smart Meter"
  - "Panel 21 refId=B (Current) migrated to CT; refId=A (Power Factor) stays on Smart Meter"
  - "Dashboard version bumped from 45 to 46"

patterns-established:
  - "CT grid table: FROM \"grid\" with clean field names — no _sensor suffix"
  - "Smart Meter stays authoritative for frequency and power_factor only"

requirements-completed:
  - GRID-01
  - GRID-02
  - OVER-04
  - OVER-05
  - OVER-06
  - EVNT-01
  - EVNT-02

duration: 15min
completed: 2026-04-04
---

# Phase 09: CT Grid Meter Migration Summary

**12 dashboard panels migrated from Smart Meter to CT grid meter — signed power enables backfeed visibility in Power Profile, voltage/current served from dedicated CT sensor**

## Performance

- **Duration:** ~15 min
- **Completed:** 2026-04-04
- **Tasks:** 2 (1 automated + 1 human checkpoint)
- **Files modified:** 1

## Accomplishments
- Migrated all 12 affected panels from `Smart Meter` table with `_sensor`-suffixed fields to `grid` CT table with clean field names
- Removed `GREATEST()` clip from Panel 801 Grid series — negative kW now visible during backfeed periods
- Panels 17/18 (Frequency/Power Factor) and mixed-source series correctly preserved on Smart Meter
- Human verification: all panels confirmed showing CT data values consistent with prior Smart Meter readings

## Task Commits

1. **Task 1: Apply 10 surgical SQL replacements** - `f5836da` (feat)
2. **Task 2: Human verification** - approved by user

## Files Created/Modified
- `solar-pv-monitor.json` — 10 SQL replacements across 12 panels, version bumped to 46

## Decisions Made
- Applied all 10 changes via Python script using `json.load`/`json.dump` for correctness on backfeed panels (33/34/35) where the JSON-escaped pattern matching differed from the in-memory string format
- Panel 34's pattern used `MIN()` not `MAX()` — fixed match condition accordingly

## Deviations from Plan

### Minor Implementation Detail
**Pattern matching for backfeed panels (Changes 8/9/10)**
- **Issue:** Plan specified patterns with `\\\"` escaping (as they appear in raw JSON file), but after earlier changes were applied via `content.replace()`, the file was re-serialized via `json.dump` which uses different escaping. The pattern `MAX(power_sensor)` also didn't match (actual SQL uses `MIN(power_sensor)` for Panel 34).
- **Fix:** Applied backfeed panel changes via `json.load` → mutate in-memory → `json.dump` for correctness.
- **Impact:** Zero — final result is identical to plan specification.

---

**Total deviations:** 1 minor implementation detail (no scope change, correct outcome)

## Issues Encountered
None — all 14 verification assertions passed.

## Next Phase Readiness
- CT grid meter data now feeds all relevant panels
- Dashboard v46 is importable and verified in live Grafana
- Ready for any further solar monitoring enhancements

---
*Phase: 09-update-panels-to-use-new-ct-meter-data-for-grid-energy-import-current-voltage-real-power-instead-of-smart-meter*
*Completed: 2026-04-04*
