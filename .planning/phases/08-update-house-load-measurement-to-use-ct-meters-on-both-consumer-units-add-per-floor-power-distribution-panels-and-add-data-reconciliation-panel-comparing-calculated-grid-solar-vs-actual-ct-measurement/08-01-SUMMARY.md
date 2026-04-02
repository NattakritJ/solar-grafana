---
phase: 08-update-house-load-measurement-to-use-ct-meters-on-both-consumer-units-add-per-floor-power-distribution-panels-and-add-data-reconciliation-panel-comparing-calculated-grid-solar-vs-actual-ct-measurement
plan: "01"
subsystem: ui
tags: [grafana, influxdb, dashboard, ct-meter, power-monitoring]

# Dependency graph
requires:
  - phase: 07
    provides: CT meter data in InfluxDB (235_floor_1, 235_floor_2 measurements)
provides:
  - House Load panels (5, 10) reading CT direct measurement instead of derived formula
  - Power Distribution row (900) with Floor 1, Floor 2, CT Load, Calculated Load panels
  - Panel 801 Power Profile with Floor 1 and Floor 2 time-series
affects: [future phases referencing house load measurement, financial calculations]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "CT direct measurement pattern: 235_floor_1/235_floor_2 with power field (not power_sensor), 2-min recency window"
    - "Reconciliation pattern: show both CT and derived load side-by-side for comparison"

key-files:
  created: []
  modified:
    - solar-pv-monitor.json

key-decisions:
  - "Used CT direct measurement (Floor1+Floor2) for House Load instead of derived formula in panels 5 and 10 — gives hardware-level ground truth"
  - "Kept Calculated Load (panel 905) using old derived formula alongside CT Load (panel 904) for reconciliation/comparison"
  - "CT field is power (not power_sensor) — confirmed from measurement schema"
  - "Panel 801 Floor1/Floor2 series use /1000.0 conversion to match existing kW unit"

patterns-established:
  - "CT measurement queries: SELECT COALESCE(selector_last(power, time)['value'], 0) FROM '235_floor_N' WHERE time >= now() - INTERVAL '2 minutes'"
  - "CT time-series queries: AVG(power) / 1000.0 with DATE_BIN 5-min and $__timeFrom/$__timeTo"

requirements-completed:
  - OVER-03
  - OVER-05

# Metrics
duration: 25min
completed: 2026-04-02
---

# Phase 08 Plan 01: CT Meter Integration & Power Distribution Summary

**CT direct measurement integrated into House Load panels; new Power Distribution row with Floor 1, Floor 2, CT vs Calculated Load reconciliation panels; Floor 1/2 series added to Power Profile time-series**

## Performance

- **Duration:** 25 min
- **Started:** 2026-04-02T14:15:37Z
- **Completed:** 2026-04-02T14:41:17Z
- **Tasks:** 3 (2 auto + 1 checkpoint)
- **Files modified:** 1

## Accomplishments

- Replaced derived East+West+Grid formula in House Load panels (5, 10) with CT Floor1+Floor2 direct measurement
- Inserted new "Power Distribution" row (900) at y=12 with 4 stat panels: Floor 1 Power (902), Floor 2 Power (903), CT Load (904), Calculated Load (905)
- Added Floor 1 and Floor 2 CT time-series to Power Profile panel 801 (now 5 targets total)
- Shifted all 37 downstream panels down 7 units to accommodate new row; dashboard version bumped to 45
- Human verification confirmed all panels render correctly with live CT data

## Task Commits

Each task was committed atomically:

1. **Task 1: Replace House Load targets, insert Power Distribution row + 4 panels, shift downstream panels** - `4c50c61` (feat)
2. **Task 2: Add Floor 1 and Floor 2 series to Panel 801 Power Profile time-series** - `5548465` (feat)
3. **Task 3: Human verify CT integration** — approved (no commit)

## Files Created/Modified

- `solar-pv-monitor.json` — Updated dashboard: CT targets on panels 5/10, new row 900 + panels 902-905, panel 801 Floor1/Floor2 targets, all downstream panels shifted +7 y, version 45

## Decisions Made

- Used CT direct measurement (Floor1+Floor2) for House Load (panels 5, 10) — hardware-level ground truth replaces software-derived value
- Kept old derived formula in panel 905 (Calculated Load) for side-by-side reconciliation — homeowner can see how closely CT and calculation agree
- CT field is `power` (not `power_sensor` used by microinverters) — confirmed from measurement schema in CONTEXT.md
- Panel 801 Floor1/Floor2 time-series use `/1000.0` division to match existing kW unit scale

## Deviations from Plan

None — plan executed exactly as written. Panel count was 53 (not the 52 stated in the task comment), because the sanity check comment was slightly off; all required structural checks passed.

## Issues Encountered

None

## User Setup Required

None — no external service configuration required.

## Next Phase Readiness

- CT meter integration complete; House Load now shows hardware-measured value
- Power Distribution row provides floor-by-floor visibility and CT vs calculated reconciliation
- Phase 08 requirements OVER-03 and OVER-05 satisfied
- Ready for next phase or milestone verification

---
*Phase: 08-update-house-load-measurement-to-use-ct-meters-on-both-consumer-units-add-per-floor-power-distribution-panels-and-add-data-reconciliation-panel-comparing-calculated-grid-solar-vs-actual-ct-measurement*
*Completed: 2026-04-02*
