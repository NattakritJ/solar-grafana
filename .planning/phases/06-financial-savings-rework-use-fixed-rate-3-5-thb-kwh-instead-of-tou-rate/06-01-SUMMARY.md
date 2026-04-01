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

key-files:
  created: []
  modified:
    - solar-pv-monitor.json

key-decisions:
  - "Replaced TOU CASE WHEN SQL (5.7982 peak / 2.6369 off-peak + 15 holiday exceptions) with SUM(hourly_kwh * 3.5) flat rate — homeowner's actual tariff is flat 3.5 THB/kWh"
  - "Removed 4 TOU breakdown panels (Peak kWh, Peak THB, Off-Peak kWh, Off-Peak THB) — they become meaningless with flat rate"
  - "Financial Savings row now shows exactly 3 panels in clean 8+8+8 wide layout"

patterns-established:
  - "Flat-rate savings SQL: SELECT SUM(hourly_kwh * 3.5) AS savings_thb FROM (DATE_BIN hourly subquery) WHERE hourly_kwh > 0"

requirements-completed: ["FINC-01", "FINC-02", "FINC-03"]

# Metrics
duration: ~2min (Task 1 complete; Task 2 awaiting human verify)
completed: "2026-04-01"
---

# Phase 06 Plan 01: Financial Savings Rework Summary

**Replaced complex TOU CASE WHEN SQL (peak/off-peak/holiday logic) with simple `SUM(hourly_kwh * 3.5)` flat-rate savings calculation across Today/Month/Year panels; removed 4 TOU-specific breakdown panels**

## Status

- **Task 1:** ✅ Complete (committed `d7ac58c`)
- **Task 2:** ⏸ Checkpoint pending — awaiting human visual verification in Grafana

## Performance

- **Duration:** ~2 min (Task 1 only; checkpoint pending for Task 2)
- **Started:** 2026-04-01T13:56:51Z
- **Completed (Task 1):** 2026-04-01T13:57:56Z
- **Tasks:** 1/2 complete (Task 2 is checkpoint:human-verify)
- **Files modified:** 1

## Accomplishments
- Replaced TOU multi-rate SQL (CASE WHEN with 15 Thai public holiday exceptions) in panels 38, 39, 40 with simple `SUM(hourly_kwh * 3.5)` — 6 rawSql values updated (refId A+B on each panel)
- Removed 4 TOU breakdown panels (41=Peak kWh, 42=Peak THB, 43=Off-Peak kWh, 44=Off-Peak THB) — dashboard now has exactly 3 Financial Savings panels
- Shifted Backfeed Log row (700) and panels 33/34/35 up by 4 grid units (y=93→89 for row, y=94→90 for panels) to fill the gap
- Expression targets (`$A + $B`) on panels 38/39/40 preserved unchanged — East+West summation still works correctly

## Task Commits

Each task was committed atomically:

1. **Task 1: Replace TOU SQL with flat-rate 3.5 THB/kWh in panels 38/39/40, remove panels 41-44** - `d7ac58c` (feat)
2. **Task 2: Verify flat-rate savings panels display correctly in Grafana** - ⏸ checkpoint pending (human-verify)

**Plan metadata:** (pending — will be added after Task 2 verification)

## Files Created/Modified
- `solar-pv-monitor.json` — Financial savings SQL simplified to flat 3.5 THB/kWh rate; TOU panels 41-44 removed; Backfeed Log row repositioned

## Decisions Made
- Used flat 3.5 THB/kWh rate — homeowner's actual electricity tariff is flat rate, not TOU. Previous complex TOU SQL was calculating incorrect savings figures.
- Removed TOU breakdown panels entirely rather than repurposing them — with flat rate, peak/off-peak distinction has no meaning.

## Deviations from Plan
None - plan executed exactly as written.

## Issues Encountered
None — Python script ran cleanly, all 5 automated verification checks passed on first run.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Task 2 (human verification) requires importing `solar-pv-monitor.json` into Grafana and confirming Financial Savings panels show plausible flat-rate THB values
- After verification approval: plan is complete, Phase 06 closes

---
*Phase: 06-financial-savings-rework-use-fixed-rate-3-5-thb-kwh-instead-of-tou-rate*
*Completed: 2026-04-01 (Task 1); Task 2 checkpoint pending*
