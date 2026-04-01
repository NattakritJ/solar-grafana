---
phase: 07-fix-stale-data-when-inverter-goes-offline-at-sunset
plan: "01"
subsystem: dashboard
tags: [grafana, influxdb-sql, time-filter, stale-data]

# Dependency graph
requires:
  - phase: 06-financial-savings-rework-use-fixed-rate-3-5-thb-kwh-instead-of-tou-rate
    provides: Dashboard with flat-rate savings panels and all prior panel structure
provides:
  - 2-minute recency window on all 53 latest-value queries across 23 panels
  - Night-time no-data behavior for offline inverter panels
affects: []

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Time-bounded latest-value: WHERE time >= now() - INTERVAL '2 minutes' ORDER BY time DESC LIMIT 1"

key-files:
  created: []
  modified:
    - solar-pv-monitor.json

key-decisions:
  - "2-minute window chosen: generous enough for 10-30s reporting gaps, tight enough to show no-data within ~2 minutes of inverter shutdown"
  - "Smart Meter queries also get 2-minute window for consistency, though it rarely goes offline"
  - "Panel 8 (System Status) excluded — already has its own 1-minute filter"

patterns-established:
  - "Latest-value queries must always include a recency WHERE clause to prevent stale data display"

requirements-completed: [OVER-01, OVER-02, OVER-04, OVER-05, PROD-01, MODL-01, MODL-02, HLTH-01, HLTH-02, HLTH-03, GRID-01]

# Metrics
duration: 3min
completed: 2026-04-01
---

# Phase 7: Fix Stale Data When Inverter Goes Offline at Sunset — Summary

**Added `WHERE time >= now() - INTERVAL '2 minutes'` to all 53 latest-value queries across 23 panels via 6 surgical string replacements so inverter panels show "no data" at night instead of frozen daytime values**

## Performance

- **Duration:** 3 min
- **Started:** 2026-04-01T14:30:00Z
- **Completed:** 2026-04-01T14:33:00Z
- **Tasks:** 2 (1 auto + 1 human-verify checkpoint)
- **Files modified:** 1

## Accomplishments
- Fixed stale-data freeze on all 23 affected panels (53 queries total)
- Inverter panels now show "no data" when inverters have been silent >2 minutes (e.g., after sunset)
- Grid quality panels (Voltage, Frequency, PF, Current) continue to show live Smart Meter data
- Panel 8 (System Status) preserved with its existing 1-minute filter — naturally excluded by replacement patterns

## Task Commits

Each task was committed atomically:

1. **Task 1: Add 2-minute recency window to all 53 latest-value queries** - `be806d1` (fix)
2. **Task 2: Human verification checkpoint** - approved by user (no code changes)

## Files Created/Modified
- `solar-pv-monitor.json` - Added `WHERE time >= now() - INTERVAL '2 minutes'` to 53 rawSql queries across 23 panels

## Decisions Made
- 2-minute window balances responsiveness (shows no-data quickly after sunset) with tolerance for temporary reporting gaps during active production
- Applied to Smart Meter queries too for consistency — Smart Meter is always online so the filter is effectively a no-op during normal operation
- Used raw text replacement (not JSON parsing) for surgical precision on escaped JSON string patterns

## Deviations from Plan
None - plan executed exactly as written

## Issues Encountered
None

## User Setup Required
None - import the updated `solar-pv-monitor.json` into Grafana as before.

## Next Phase Readiness
- Phase 7 is the final planned phase — all v1.0 milestone work complete
- Dashboard ready for production use with correct night-time behavior

---
*Phase: 07-fix-stale-data-when-inverter-goes-offline-at-sunset*
*Completed: 2026-04-01*
