---
phase: 01-foundation-overview-stats
plan: "02"
subsystem: infra
tags: [grafana, influxdb, sql, dashboard, stat-panels, power-flow]

# Dependency graph
requires:
  - phase: 01-foundation-overview-stats plan 01
    provides: Dashboard JSON skeleton with __inputs, 7 section rows, 3 overview stat panels
provides:
  - 5 additional overview stat panels completing the KPI row (Grid Import, House Load, Self-Consumption, Peak Today, System Status)
  - Power flow visualization row with 3 stat panels (Solar → House ← Grid)
  - Complete Phase 1 dashboard with all 9 OVER requirements and 3 power flow panels
affects:
  - All subsequent phases (panels build on this dashboard JSON)
  - Phase 2 (production charts will use same SQL patterns established here)

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Cross-join subquery pattern for combining data from 3 measurements (East + West + Smart Meter)"
    - "GREATEST(value, 0) clamping pattern to prevent negative house load values"
    - "CASE WHEN division-by-zero guard for ratio calculations (self-consumption)"
    - "LEAST/GREATEST clamping for percentage values to 0-1 range"
    - "Value mappings for device state codes (Standby/Normal/Warning/Fault/Offline)"
    - "colorMode=background for power flow panels vs colorMode=value for KPI panels"

key-files:
  created: []
  modified:
    - solar-pv-monitor.json

key-decisions:
  - "Used cross-join subquery approach for House Load and Self-Consumption (combines 3 measurements in single query)"
  - "Self-Consumption outputs 0-1 range with percentunit (not 0-100 with percent) per D-15"
  - "Peak Today uses sum-of-individual-peaks approximation (simpler than JOIN across measurements)"
  - "System Status queries East Microinverter only (primary), uses value mappings for state codes 0-3 + null=Offline"
  - "Power flow panels use colorMode=background and graphMode=area to visually distinguish from KPI stats row"

patterns-established:
  - "Pattern 5: Cross-join subqueries for multi-measurement calculations — FROM (subquery) AS east, (subquery) AS west, (subquery) AS grid"
  - "Pattern 6: GREATEST(..., 0) clamping for derived values that should never be negative"
  - "Pattern 7: CASE WHEN guard for division — check denominator = 0 before dividing"
  - "Pattern 8: Value mappings array for device state sensor codes in stat panels"
  - "Pattern 9: colorMode=background + graphMode=area for power flow visualization stat panels"

requirements-completed: [OVER-04, OVER-05, OVER-06, OVER-07, OVER-08, OVER-09]

# Metrics
duration: 3min
completed: "2026-03-30"
---

# Phase 01 Plan 02: Remaining Overview Stats & Power Flow Summary

**5 KPI stat panels (Grid Import, House Load, Self-Consumption %, Peak Today, System Status) and 3 power flow panels (Solar → House ← Grid) completing all Phase 1 overview requirements**

## Performance

- **Duration:** ~3 min
- **Started:** 2026-03-30T04:10:44Z
- **Completed:** 2026-03-30T04:13:13Z
- **Tasks:** 2 completed
- **Files modified:** 1

## Accomplishments
- Added 5 KPI stat panels completing the 8-panel overview row: Grid Import (blue), House Load (orange), Self-Consumption % (green gradient), Peak Today (solar gradient), System Status (value-mapped states)
- Added 3 power flow stat panels forming a visual "Solar → House ← Grid" energy direction display with background coloring and area sparklines
- All SQL queries use COALESCE for NULL handling, double-quoted measurement names, and avoid time-range macros (deferred to Phase 2)
- House Load correctly sums solar + grid import with GREATEST(0) clamping per Pitfall 5
- Self-Consumption uses CASE WHEN to avoid division by zero, outputs 0-1 range for percentunit display

## Task Commits

Each task was committed atomically:

1. **Task 1: Add Grid Import, House Load, Self-Consumption, Peak Power, and System Status stat panels** - `f62494f` (feat)
2. **Task 2: Add Power Flow row with 3 stat panels (Solar → House ← Grid)** - `4e663f3` (feat)

**Plan metadata:** _(final docs commit — see below)_

## Files Created/Modified
- `solar-pv-monitor.json` — Updated Grafana dashboard JSON: now contains 11 stat panels (8 KPI + 3 power flow), 7 row sections, all Phase 1 OVER requirements implemented

## Decisions Made
- **Cross-join subqueries for multi-source calculations:** House Load and Self-Consumption panels combine data from East Microinverter, West Microinverter, and Smart Meter using `FROM (subquery) AS east, (subquery) AS west, (subquery) AS grid` cross-join pattern. This is the simplest single-query approach; if InfluxDB 3 doesn't support cross-joins, scalar subselects or multi-query+transformation is the fallback.
- **Self-Consumption in 0-1 range with percentunit:** Per D-15, SQL outputs 0-1 (not 0-100) so Grafana's percentunit formatter displays correctly. Thresholds at 0.3/0.7/1.0.
- **Peak Today as sum of individual peaks (approximation):** Sum of MAX(power_sensor) from each inverter separately, not peak of simultaneous sum. Acceptable for Phase 1; true simultaneous peak would require JOIN on time-binned data.
- **System Status from East Microinverter only:** Per CONTEXT D-67 (agent's discretion), East inverter is primary status source. Value mappings cover states 0 (Standby), 1 (Normal), 2 (Warning), 3 (Fault), null (Offline).
- **Power flow visual distinction:** colorMode=background + graphMode=area vs colorMode=value + graphMode=none separates the power flow row from KPI row visually.

## Deviations from Plan

None — plan executed exactly as written.

## Issues Encountered

None — all queries follow InfluxDB 3 Core SQL syntax. JSON validated successfully. All 11 stat panels verified.

## User Setup Required

None - dashboard is self-contained. On import into Grafana, user will be prompted to map `DS_INFLUXDB_SOLAR` to their `influxdb-solar` datasource.

## Known Stubs

None — all 11 stat panels have complete SQL queries targeting real InfluxDB measurements. No placeholder data or hardcoded mock values. Panels will show "No Data" if InfluxDB is unreachable, which is correct behavior.

## Next Phase Readiness
- Phase 1 is now complete — all INFR (01-06) and OVER (01-09) requirements addressed
- `solar-pv-monitor.json` has 11 stat panels + 7 row sections ready for Phase 2 to add time-series charts
- SQL patterns established: UNION ALL aggregation, cross-join multi-measurement, COALESCE null handling, GREATEST clamping, CASE WHEN division guard
- Phase 2 will introduce `$__timeFrom`/`$__timeTo` time-range macros for time-series panels — the first real validation of macro substitution in InfluxDB 3 SQL mode

---
*Phase: 01-foundation-overview-stats*
*Completed: 2026-03-30*
