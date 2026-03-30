---
phase: 03-module-level-detail-inverter-health-event-log
plan: 01
subsystem: dashboard
tags: [grafana, bargauge, table, module-level, solar-panels]

requires:
  - phase: 02-production-charts-grid-monitoring
    provides: Dashboard with overview stats, production charts, grid monitoring panels (IDs 1-21)
provides:
  - Per-panel power bar gauge (8 panels, horizontal bars with production color scale)
  - Module detail table with power, voltage, current, today kWh, total kWh per panel
  - Uncollapsed Module Level row (id=300) with both panels
affects: [03-02, 03-03, phase-4-canvas-layout]

tech-stack:
  added: [table panel type]
  patterns: [8-query merge for per-panel data, COALESCE for nighttime NULLs on pv fields]

key-files:
  created: []
  modified: [solar-pv-monitor.json]

key-decisions:
  - "Panel labels use 'East PV1'...'West PV4' naming for clarity (D-26)"
  - "Production color scale: dark-gray 0W, dark-amber 1-100W, amber 100-400W, green 400-600W, dark-green 600W+ (D-27)"
  - "East panels orange (#FF9830), West panels blue (#5794F2) color overrides (D-48)"
  - "655W max on bar gauge matching individual panel rating"

patterns-established:
  - "Per-panel queries: 8 separate queries with merge transformation, one per PV channel per inverter"
  - "Table panel with column-level overrides for units, decimals, and color thresholds"

requirements-completed: [MODL-01, MODL-02, MODL-03]

duration: 3min
completed: 2026-03-30
---

# Phase 03 Plan 01: Module-Level Bar Gauge and Detail Table Summary

**Per-panel monitoring with 8-panel bar gauge and comprehensive detail table showing power, voltage, current, today/total kWh for all panels**

## Performance

- **Duration:** 3 min
- **Started:** 2026-03-30T05:42:03Z
- **Completed:** 2026-03-30T05:45:00Z
- **Tasks:** 2/2
- **Files modified:** 1

## Accomplishments

### Task 1: Uncollapse Module Level row and add per-panel power bar gauge
- Set Row 300 (Module Level) to `collapsed: false`
- Added Panel 22 (bargauge): "Panel Power" — 8 horizontal bars for East PV1-4 and West PV1-4
- 8 separate SQL queries using `COALESCE(pvN_power_sensor, 0)` with `ORDER BY time DESC LIMIT 1`
- Threshold color scale: #1a1a2e (0W), #614a19 (1W), #c7a035 (100W), #73BF69 (400W), #1a7c11 (600W)
- East panels use orange (#FF9830) overrides, West panels use blue (#5794F2)
- Added `table` panel type to `__requires`
- Adjusted downstream row y-positions

### Task 2: Add module-level detail table
- Added Panel 23 (table): "Module Detail" — 8 rows with Panel, Power (W), Voltage (V), Current (A), Today (kWh), Total (kWh)
- 8 queries selecting all per-panel fields with COALESCE wrapping
- Merge transformation combines all 8 query results into rows
- Column overrides: Power has color-background, Voltage in volt, Current in amp, Today/Total in kwatth
- Hidden `time` column via override
- Sorted by Panel name ascending

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| 1+2 | 1c0ff6d | feat(03-01): add module-level bar gauge and detail table panels |

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — all panels are wired to live InfluxDB queries.

## Self-Check: PASSED
