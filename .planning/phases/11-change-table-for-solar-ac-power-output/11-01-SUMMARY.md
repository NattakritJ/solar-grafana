---
phase: 11-change-table-for-solar-ac-power-output
plan: "01"
subsystem: dashboard
tags: [grafana, influxdb, sql, ct-sensor, real-time]

# Dependency graph
requires:
  - phase: 10-avg-windowed-aggregates
    provides: "AVG windowed aggregate pattern for all snapshot queries"
  - phase: 09-ct-grid-migration
    provides: "CT table migration precedent (grid table, unquoted names, no _sensor suffix)"
  - phase: 08-ct-house-load
    provides: "CT field naming convention (power, voltage, current — no _sensor suffix)"
provides:
  - "8 solar AC power panels reading from fast CT sensors (east/west_microinverter_power) instead of slow inverter tables"
  - "All 15 CT snapshot panels using 10-second recency windows for faster updates"
  - "5-second dashboard refresh interval matching fast CT data rate"
affects: []

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "CT sensor tables for solar AC power: east_microinverter_power / west_microinverter_power"
    - "10-second recency window for all CT snapshot panels (was 1 minute)"
    - "5-second dashboard refresh (was 10 seconds)"

key-files:
  created: []
  modified:
    - "solar-pv-monitor.json"

key-decisions:
  - "CT tables unquoted (no spaces) — east_microinverter_power not \"east_microinverter_power\""
  - "power field (no _sensor suffix) — consistent with existing CT table convention from Phase 8/9"
  - "10-second recency window for all CT tables — captures ~5 readings at 2s intervals"
  - "5-second refresh interval — balances responsiveness with browser/server load"
  - "Inverter-only panels unchanged at 1-minute window — inverter logs at ~60s intervals"

patterns-established:
  - "CT solar AC power queries: SELECT COALESCE(AVG(power), 0) FROM east_microinverter_power WHERE time >= now() - INTERVAL '10 seconds'"
  - "All CT snapshot panels: 10-second recency window"

requirements-completed: [OVER-01, OVER-05, OVER-06, OVER-09, PROD-01, PROD-02, GRID-01]

# Metrics
duration: 3min
completed: 2026-04-09
---

# Phase 11 Plan 01: Change Table for Solar AC Power Output Summary

**Migrated 8 solar AC panels from slow-polling inverter tables to fast CT sensors, tightened all 15 CT snapshot panels to 10-second recency windows, and set 5s dashboard refresh**

## Performance

- **Duration:** 3 min
- **Started:** 2026-04-09T00:19:15Z
- **Completed:** 2026-04-09T00:22:15Z
- **Tasks:** 1 of 1 auto tasks complete (Task 2 is human-verify checkpoint)
- **Files modified:** 1

## Accomplishments
- 8 panels migrated from `"East/West Microinverter"` → `east/west_microinverter_power` CT tables (16 SQL replacements)
- All 28 CT snapshot targets across 15 panels tightened from `INTERVAL '1 minutes'` to `INTERVAL '10 seconds'`
- Dashboard refresh changed from 10s to 5s, version bumped 46 → 47
- Inverter-only panels (DC channels, temperature, state, energy counters) verified unchanged

## Task Commits

Each task was committed atomically:

1. **Task 1: Migrate 8 panels from inverter to CT tables and tighten CT recency windows** - `022d23b` (feat)

**Plan metadata:** (pending — docs commit after state updates)

## Files Created/Modified
- `solar-pv-monitor.json` - Dashboard JSON with 8 panels migrated to CT tables, 28 CT targets tightened to 10s window, 5s refresh

## Decisions Made
- CT table names are unquoted (no spaces) — `east_microinverter_power` not `"east_microinverter_power"`, consistent with Phase 8/9 convention
- Field name `power` (no `_sensor` suffix) — consistent with existing CT field convention
- 10-second recency window for all CT tables — captures ~5 readings at 2s log intervals, sufficient for AVG smoothing while being much more responsive than old 1-minute window
- 5-second refresh to take advantage of fast CT data rate
- Inverter-only panels (23-31, etc.) kept at 1-minute window since inverter still logs at ~60s intervals

## Deviations from Plan

None — plan executed exactly as written. All 30 replacements (12 Part A + 18 Part B) applied successfully with verification passing on first run.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Known Stubs
None — all queries wired to live CT data tables.

## Next Phase Readiness
- Dashboard ready for live verification (Task 2 checkpoint)
- CT sensor data confirmed live at ~2s intervals since 2026-04-08T23:26:45
- Future consideration: migrate energy panels to CT `energy` field once daily-reset behavior is confirmed over multiple days

---
*Phase: 11-change-table-for-solar-ac-power-output*
*Completed: 2026-04-09*
