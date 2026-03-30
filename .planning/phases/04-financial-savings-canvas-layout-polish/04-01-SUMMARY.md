---
phase: 04-financial-savings-canvas-layout-polish
plan: 01
subsystem: financial
tags: [tou, savings, thai-holidays, influxdb-sql, stat-panel, currency-thb]

# Dependency graph
requires:
  - phase: 03-module-level
    provides: Dashboard with 37 panels, 7 section rows, established stat panel patterns
provides:
  - 7 TOU-aware financial savings stat panels (ids 38-44)
  - Today/MTD/YTD savings summary in THB
  - Peak vs off-peak kWh and THB breakdown
  - Thai public holidays classified as off-peak
affects: [04-03-polish, dashboard-final]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "TOU classification via CASE WHEN with DOW + HOUR extraction AT TIME ZONE 'Asia/Bangkok'"
    - "Thai public holidays hard-coded as 15 dates for 2026 in SQL WHERE clauses"
    - "Hourly energy estimation: DATE_BIN(1 hour) + AVG(power)/1000 = kWh per hour"
    - "Currency display: Grafana custom unit 'suffix: THB' for Thai Baht formatting"
    - "Peak/off-peak filter inversion: AND NOT (holiday) for peak, OR (holiday) for off-peak"

key-files:
  created: []
  modified:
    - solar-pv-monitor.json

key-decisions:
  - "Hourly bucket approach for TOU energy: AVG(power)/1000 per DATE_BIN hour gives kWh without needing counter deltas"
  - "Thai holidays encoded as month/day extraction (not date strings) for InfluxDB 3 Core SQL compatibility"
  - "Panels 38-40 positioned at y:82 (not y:64 as planned) due to Canvas+Heatmap panels inserted by parallel 04-02 agent"
  - "Currency formatting uses Grafana 'suffix: THB' custom unit — confirmed working"

patterns-established:
  - "TOU SQL pattern: Weekday check (DOW NOT IN 0,6) + hour check (BETWEEN 9 AND 21) + holiday exclusions"
  - "Multi-inverter financial aggregation: 2 queries (East+West) + merge + reduce(sum) transformations"

requirements-completed: [FINC-01, FINC-02, FINC-03, FINC-04]

# Metrics
duration: 19min
completed: 2026-03-30
---

# Phase 4 Plan 01: TOU-Aware Financial Savings Panels Summary

**7 stat panels for financial savings with full TOU classification including Thai public holidays, peak/off-peak breakdown, and Today/MTD/YTD cumulative views**

## Performance

- **Duration:** 19 min
- **Started:** 2026-03-30T06:51:23Z
- **Completed:** 2026-03-30T07:10:57Z
- **Tasks:** 2 completed
- **Files modified:** 1

## Accomplishments

- Added 7 financial savings stat panels (ids 38-44) under the Financial Savings row (id=600)
- Today's Savings, Month Savings, and Year Savings show cumulative THB using TOU-classified SQL
- Peak kWh, Peak THB, Off-Peak kWh, and Off-Peak THB provide granular TOU breakdown
- All SQL queries use `AT TIME ZONE 'Asia/Bangkok'` for correct TOU period classification
- All 15 Thai public holidays for 2026 are hard-coded as off-peak in every query
- Peak rate: 5.7982 THB/kWh (weekday 09-22), Off-Peak rate: 2.6369 THB/kWh

## Task Commits

Each task was committed atomically:

1. **Task 1: Uncollapse Financial Savings row and add 3 summary savings stat panels** - `40043a2` (feat)
2. **Task 2: Add 4 peak/off-peak breakdown stat panels** - `185152d` (feat)

## Files Created/Modified

- `solar-pv-monitor.json` — Added 7 financial stat panels (38-44), uncollapsed Row 600, repositioned Row 700 and panels 33-37

## Panel Layout

| Panel ID | Title | gridPos | Unit |
|----------|-------|---------|------|
| 38 | Today's Savings | y:82, x:0, w:8, h:4 | suffix: THB |
| 39 | Month Savings | y:82, x:8, w:8, h:4 | suffix: THB |
| 40 | Year Savings | y:82, x:16, w:8, h:4 | suffix: THB |
| 41 | Peak kWh | y:86, x:0, w:6, h:4 | kwatth |
| 42 | Peak THB | y:86, x:6, w:6, h:4 | suffix: THB |
| 43 | Off-Peak kWh | y:86, x:12, w:6, h:4 | kwatth |
| 44 | Off-Peak THB | y:86, x:18, w:6, h:4 | suffix: THB |

## SQL Query Pattern (TOU Classification)

All financial panels use this hourly-bucket TOU approach:
```sql
SELECT SUM(CASE
  WHEN EXTRACT(DOW FROM bucket AT TIME ZONE 'Asia/Bangkok') IN (0, 6) THEN hourly_kwh * 2.6369
  WHEN [holiday exclusions] THEN hourly_kwh * 2.6369
  WHEN EXTRACT(HOUR FROM bucket AT TIME ZONE 'Asia/Bangkok') >= 9
   AND EXTRACT(HOUR FROM bucket AT TIME ZONE 'Asia/Bangkok') < 22 THEN hourly_kwh * 5.7982
  ELSE hourly_kwh * 2.6369
END) AS savings_thb
FROM (
  SELECT DATE_BIN(INTERVAL '1 hour', time, '1970-01-01T00:00:00Z') AS bucket,
    AVG(pv_power_sum) / 1000.0 AS hourly_kwh
  FROM "[East/West] Microinverter"
  WHERE time >= [time_range]
  GROUP BY 1
) WHERE hourly_kwh > 0
```

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Y-position conflict from parallel agent execution**
- **Found during:** Task 1
- **Issue:** Parallel agent (04-02) had already inserted Canvas roof layout (h:10) and daily heatmap (h:8) panels above the Financial Savings row, shifting all Y positions. 04-02 also pre-created panels 38-40 but at incorrect Y positions (y:72 instead of y:82).
- **Fix:** Recalculated Y positions: Row 600 at y:81, panels 38-40 moved to y:82, panels 41-44 at y:86, Row 700 shifted to y:90
- **Files modified:** solar-pv-monitor.json
- **Commit:** 40043a2

**2. [Rule 2 - Missing functionality] Panels 38-40 already existed but at wrong positions**
- **Found during:** Task 1
- **Issue:** The parallel 04-02 agent had created panels 38-40 with correct TOU SQL and holiday logic, but placed them at y:72 (not accounting for the Canvas panel height). These needed repositioning, not recreation.
- **Fix:** Repositioned panels 38-40 from y:72 to y:82 to sit correctly below Row 600 (y:81)
- **Files modified:** solar-pv-monitor.json
- **Commit:** 40043a2

## Known Stubs

None — all panels have complete SQL queries with real data sources, TOU rates, and holiday logic.

## Self-Check: PASSED

- ✅ solar-pv-monitor.json exists
- ✅ 04-01-SUMMARY.md exists
- ✅ Commit 40043a2 (Task 1) found
- ✅ Commit 185152d (Task 2) found
