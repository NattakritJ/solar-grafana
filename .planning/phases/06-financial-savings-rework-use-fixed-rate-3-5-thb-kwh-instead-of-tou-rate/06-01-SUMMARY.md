---
phase: 06-financial-savings-rework-use-fixed-rate-3-5-thb-kwh-instead-of-tou-rate
plan: "01"
subsystem: dashboard
tags: [grafana, influxdb, sql, financial-savings, flat-rate]

# Dependency graph
requires:
  - phase: 05-fix-all-panel-in-dashboard-that-still-use-transformation-to-calcuate-to-use-expression-instead-for-example-look-at-self-consumption-or-house-load-panel-that-use-expression
    provides: Expression targets on financial savings panels 38-44; no more transformation chains
provides:
  - Financial savings panels 38/39/40 using flat 3.5 THB/kWh rate (no TOU complexity)
  - TOU breakdown panels 41-44 removed from dashboard
  - Clean Financial Savings row with exactly 3 panels (Today/Month/Year)
  - Backfeed Log row shifted up by 4 grid units to fill gap
affects: []

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Flat-rate kWh savings: SUM(hourly_kwh * 3.5) pattern for all savings panels"
    - "Hourly energy buckets: DATE_BIN 1-hour + AVG(sum of PV channels) / 1000 = kWh"
    - "Bangkok timezone boundary: date_trunc('period', now() AT TIME ZONE 'Asia/Bangkok') AT TIME ZONE 'UTC'"

key-files:
  created: []
  modified:
    - solar-pv-monitor.json

key-decisions:
  - "Replaced TOU CASE WHEN SQL (5.7982 peak / 2.6369 off-peak + 15 holiday exceptions) with SUM(hourly_kwh * 3.5) flat rate — homeowner's actual tariff is flat 3.5 THB/kWh"
  - "Removed 4 TOU breakdown panels (Peak kWh, Peak THB, Off-Peak kWh, Off-Peak THB) — they become meaningless with flat rate"
  - "Bangkok timezone applied to all three savings boundaries (today/month/year) for consistent calendar-day scoping"
  - "Financial Savings row now shows exactly 3 panels in clean 8+8+8 wide layout"

patterns-established:
  - "Flat-rate savings SQL: SELECT SUM(hourly_kwh * 3.5) AS savings_thb FROM (DATE_BIN hourly subquery) WHERE hourly_kwh > 0"

requirements-completed: ["FINC-01", "FINC-02", "FINC-03"]

# Metrics
duration: ~15min (Task 1 + fix + Task 2 human verification)
completed: "2026-04-01"
---

# Phase 06 Plan 01: Financial Savings Rework Summary

**Replaced complex TOU CASE WHEN SQL (peak/off-peak/holiday logic) with simple `SUM(hourly_kwh * 3.5)` flat-rate savings calculation across Today/Month/Year panels; removed 4 TOU-specific breakdown panels; human-verified in Grafana**

## Status

- **Task 1:** ✅ Complete (committed `d7ac58c`)
- **Task 1b:** ✅ Complete — Bangkok timezone fix (committed `f50a84f`, deviation auto-fix)
- **Task 2:** ✅ Complete — human visual verification approved

## Performance

- **Duration:** ~15 min
- **Started:** 2026-04-01T13:56:51Z
- **Completed:** 2026-04-01T21:05:35Z
- **Tasks:** 2/2 complete
- **Files modified:** 1

## Accomplishments

- Replaced TOU multi-rate SQL (CASE WHEN with 15 Thai public holiday exceptions) in panels 38, 39, 40 with simple `SUM(hourly_kwh * 3.5)` — 6 rawSql values updated (refId A+B on each panel)
- Removed 4 TOU breakdown panels (41=Peak kWh, 42=Peak THB, 43=Off-Peak kWh, 44=Off-Peak THB) — dashboard now has exactly 3 Financial Savings panels
- Shifted Backfeed Log row (700) and panels 33/34/35 up by 4 grid units (y=93→89 for row, y=94→90 for panels) to fill the gap
- Expression targets (`$A + $B`) on panels 38/39/40 preserved unchanged — East+West summation still works correctly
- Applied Bangkok timezone to all three savings time boundaries for consistent calendar-day scoping

## Task Commits

Each task was committed atomically:

1. **Task 1: Replace TOU SQL with flat-rate 3.5 THB/kWh in panels 38/39/40, remove panels 41-44** — `d7ac58c` (feat)
2. **Task 1b: Fix Bangkok timezone on month/year savings boundaries** — `f50a84f` (fix, deviation)
3. **Task 2: Verify flat-rate savings panels display correctly in Grafana** — human-approved ✅

## Files Created/Modified

- `solar-pv-monitor.json` — Financial savings SQL simplified to flat 3.5 THB/kWh rate; TOU panels 41-44 removed; Bangkok timezone applied to all three boundaries; Backfeed Log row repositioned

## Decisions Made

- Used flat 3.5 THB/kWh rate — homeowner's actual electricity tariff is flat rate, not TOU. Previous complex TOU SQL was calculating incorrect savings figures.
- Removed TOU breakdown panels entirely rather than repurposing them — with flat rate, peak/off-peak distinction has no meaning.
- Applied Bangkok timezone to all three time boundaries (today/month/year) for consistent calendar-day alignment.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 2 - Missing Critical Functionality] Bangkok timezone missing from Month and Year savings boundaries**

- **Found during:** Post-Task-1 checkpoint review by orchestrator
- **Issue:** Panel 39 (Month Savings) used `date_trunc('month', now())` and Panel 40 (Year Savings) used `date_trunc('year', now())` without timezone — both evaluated in UTC. This caused incorrect boundary alignment: the Bangkok calendar month starts 7 hours before the UTC month boundary, so on the first day of any month, Today's Savings would show higher than Month Savings (Today's kWh from 00:00 Bangkok not yet counted in the month total that starts at 07:00 Bangkok / 00:00 UTC).
- **Fix:** Applied the same pattern established in Phase 05.1 — `date_trunc('month', now() AT TIME ZONE 'Asia/Bangkok') AT TIME ZONE 'UTC'` and `date_trunc('year', now() AT TIME ZONE 'Asia/Bangkok') AT TIME ZONE 'UTC'`
- **Files modified:** `solar-pv-monitor.json` (panels 39 refId=A/B, 40 refId=A/B)
- **Commit:** `f50a84f`

## Verification Results

All automated checks passed:

- ✅ TOU panels 41–44 not present in panels array
- ✅ Panels 38/39/40 rawSql contains `* 3.5`; no `CASE WHEN`, `5.7982`, or `2.6369`
- ✅ Expression targets (`$A + $B`) intact on all three savings panels
- ✅ Row 700 (Backfeed Log) at y=89
- ✅ Dashboard JSON structure valid
- ✅ Human visual approval: savings panels show plausible flat-rate THB values in Grafana

## Known Stubs

None — all three savings panels are fully wired to live InfluxDB data with correct flat-rate SQL.

## Self-Check: PASSED

- `solar-pv-monitor.json` — FOUND (modified in place)
- `d7ac58c` — FOUND (feat: replace TOU SQL with flat-rate 3.5 THB/kWh, remove panels 41-44)
- `f50a84f` — FOUND (fix: apply Bangkok timezone to month/year savings boundaries)

---
*Phase: 06-financial-savings-rework-use-fixed-rate-3-5-thb-kwh-instead-of-tou-rate*
*Completed: 2026-04-01*
