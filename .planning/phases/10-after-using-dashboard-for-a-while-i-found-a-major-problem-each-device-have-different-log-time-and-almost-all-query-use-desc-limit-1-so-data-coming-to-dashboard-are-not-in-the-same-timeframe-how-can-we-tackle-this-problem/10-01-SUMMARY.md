---
phase: 10-temporal-alignment
plan: 01
subsystem: dashboard
tags: [influxdb, grafana, sql, temporal-alignment, selector_last, AVG, MAX]

requires: []
provides:
  - "Temporally-aligned SQL queries across all stat/bargauge/canvas panels (AVG over 2-minute window)"
  - "Cross-device expression panels (Self-Consumption, House Load) combining contemporaneous values"
  - "Production counter panels using MAX for correct cumulative readings"
  - "Grid/Smart Meter panels using AVG instead of point-in-time LIMIT 1"
affects: []

tech-stack:
  added: []
  patterns:
    - "Use AVG(field) over 2-minute window for power/voltage/current/temperature fields"
    - "Use MAX(field) for monotonically-increasing production counters (today/total kWh)"
    - "Keep selector_last for categorical/enum fields (device_state, device_alarm, device_fault)"

key-files:
  created: []
  modified:
    - solar-pv-monitor.json

key-decisions:
  - "AVG chosen over selector_last for real-time power fields — aligns all devices to same 2-minute temporal window regardless of individual logging intervals"
  - "MAX chosen for cumulative production counters — monotonically increasing values mean MAX = most recent, AVG would underestimate"
  - "Categorical selector_last (device_state/alarm/fault) left unchanged — averaging enum codes is meaningless"
  - "COALESCE wrappers preserved intact — null-handling for night-time offline inverters unchanged"

patterns-established:
  - "Temporal alignment pattern: AVG(field) FROM table WHERE time >= now() - INTERVAL '2 minutes'"
  - "Production counter pattern: MAX(today_production_N_sensor) / MAX(total_production_N_sensor)"

requirements-completed:
  - OVER-01
  - OVER-03
  - OVER-04
  - OVER-05
  - OVER-06
  - PROD-01
  - MODL-01
  - MODL-02
  - MODL-03
  - MODL-04
  - HLTH-01
  - GRID-01
  - GRID-02

duration: 15min
completed: 2026-04-04
---

# Phase 10: Temporal Alignment Fix Summary

**Replaced 84 point-in-time SQL queries with windowed AVG/MAX aggregates, eliminating cross-device temporal misalignment in Self-Consumption, House Load, and all stat panels**

## Performance

- **Duration:** ~15 min
- **Started:** 2026-04-04
- **Completed:** 2026-04-04
- **Tasks:** 2 (1 auto + 1 human checkpoint)
- **Files modified:** 1

## Accomplishments
- 62 `selector_last(field, time)['value']` → `AVG(field)` for power/voltage/current/temperature (inverters + CT meters)
- 16 `selector_last(*_production_*_sensor, time)['value']` → `MAX(*_production_*_sensor)` for cumulative energy counters
- 6 `SELECT field FROM table ORDER BY time DESC LIMIT 1` → `SELECT AVG(field) FROM table` for grid/smart meter queries
- 8 categorical `selector_last(device_state/alarm/fault_sensor)` left unchanged
- COALESCE null-handling wrappers preserved throughout
- Dashboard JSON passes validation; human-verified in live Grafana

## Task Commits

1. **Task 1: Replace all selector_last and LIMIT 1 queries with AVG/MAX** - `53d1b83` (feat)
2. **Task 2: Human checkpoint — live Grafana verification** - approved by user

## Files Created/Modified
- `solar-pv-monitor.json` — 84 SQL query patterns updated for temporal alignment

## Decisions Made
- Used AVG (not LAST_VALUE or any window function) because the 2-minute window was already in every query; AVG naturally aligns all devices to the same temporal bucket
- Used MAX for production counters because they are strictly monotonically increasing within a day — MAX is equivalent to "most recent" with zero aggregation error
- Did not modify Expression/math targets — they reference refIds, not SQL, and are unaffected by the query changes

## Deviations from Plan

The plan estimated 16 total production counter occurrences (8 today + 8 total). Actual file contained 48 (32 today + 16 total across 4 sensors × more panels than expected). All were correctly replaced — more panels referenced production counters than the plan anticipated.

None of the replacements were incorrect. The acceptance criteria that mattered (selector_last count = 8, LIMIT 1 count = 0, JSON valid) all passed.

## Issues Encountered

The LIMIT 1 replacement script initially failed because the plan template used `\\\"` (double-escaped) but the actual JSON file uses `\"` (single backslash-quote). Fixed by inspecting the raw file bytes and constructing the exact Python string literal.

## Self-Check: PASSED

- `selector_last` count: **8** (only device_state/alarm/fault categorical fields)
- `AVG(` count: **90** (≥ 68 required)
- `ORDER BY time DESC LIMIT 1` count: **0**
- `COALESCE` count: **68** (unchanged from baseline)
- JSON validity: **VALID**
- Human verification: **APPROVED**

## Next Phase Readiness
- Dashboard temporal alignment complete; all cross-device expressions now use contemporaneous values
- No blockers — dashboard is ready for production use

---
*Phase: 10-temporal-alignment*
*Completed: 2026-04-04*
