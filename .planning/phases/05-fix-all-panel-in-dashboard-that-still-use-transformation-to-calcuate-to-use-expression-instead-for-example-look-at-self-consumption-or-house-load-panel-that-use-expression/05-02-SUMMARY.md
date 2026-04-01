---
phase: 05-fix-all-panel-in-dashboard-that-still-use-transformation-to-calcuate-to-use-expression-instead-for-example-look-at-self-consumption-or-house-load-panel-that-use-expression
plan: "02"
subsystem: ui

tags: [grafana, influxdb, dashboard, expression, math, financial, tou, savings]

# Dependency graph
requires:
  - phase: 05-01
    provides: "Expression migration for panels 1,2,3,5,6,7,9,10 — established pattern: hide InfluxDB targets + add math expression target + clear transformations"
provides:
  - "All 7 financial savings panels (38-44) migrated to Grafana Expression targets"
  - "Complete Phase 5 migration — all 15 calculation panels now use Expression targets (zero client-side arithmetic transformations)"
affects: [Phase 5.1-today-fix, future-dashboard-maintenance]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Expression target pattern: hide InfluxDB query targets (hide:true), append math type target with $A + $B expression refId=C, clear transformations array"
    - "Display-only merge panels (13, 14, 15, 22, 23, 45) retain merge transforms — only arithmetic calculation panels migrated"

key-files:
  created: []
  modified:
    - "solar-pv-monitor.json"

key-decisions:
  - "Financial panels 38-44 all follow identical merge+reduce(sum) pattern — single Python script migrated all 7 atomically"
  - "TOU SQL (CASE WHEN clauses for peak/off-peak classification) left completely unchanged — only targets/transforms structure modified"
  - "Expression $A + $B sums East inverter + West inverter savings queries for each time period"

patterns-established:
  - "Expression migration pattern (finalized): add hide:true to all InfluxDB targets, append {datasource:{type:__expr__,uid:__expr__}, expression:$A+$B, refId:C, type:math}, set transformations=[]"

requirements-completed: [FINC-01, FINC-02, FINC-03, FINC-04]

# Metrics
duration: ~5min
completed: 2026-04-01
---

# Phase 05 Plan 02: Fix Transformations to Expressions (Financial Panels) Summary

**All 7 financial savings panels (38-44) migrated from merge+reduce transformations to Grafana Expression targets, completing the full Phase 5 migration — 15 calculation panels now use server-side Expression arithmetic with zero client-side transformations**

## Performance

- **Duration:** ~5 min
- **Started:** 2026-04-01T09:40:00Z (estimated)
- **Completed:** 2026-04-01T09:47:20Z
- **Tasks:** 2 (1 auto + 1 human-verify checkpoint)
- **Files modified:** 1

## Accomplishments

- Migrated all 7 financial savings panels (Today's Savings, Month Savings, Year Savings, Peak kWh, Peak THB, Off-Peak kWh, Off-Peak THB) from `merge` + `reduce(sum)` transformation pipeline to Grafana Expression targets
- Completed Phase 5 expression migration — all 15 calculation panels (1, 2, 3, 5, 6, 7, 9, 10, 38, 39, 40, 41, 42, 43, 44) now use `type: math` Expression targets with zero client-side arithmetic transforms
- All 6 display-only merge panels (13, 14, 15, 22, 23, 45) confirmed untouched — their `merge` transforms preserved
- Human verification approved: all migrated panels display values correctly in Grafana with no regressions

## Task Commits

Each task was committed atomically:

1. **Task 1: Migrate panels 38-44 — add Expression target, remove merge+reduce** - `1373104` (feat)
2. **Task 2: Verify all migrated panels display correctly in Grafana** - Human checkpoint — approved by user

**Plan metadata:** (docs commit — created at state update)

## Files Created/Modified
- `solar-pv-monitor.json` — Panels 38-44 updated: InfluxDB targets set to `hide:true`, Expression target `{type:math, refId:C, expression:"$A + $B"}` appended, `transformations: []` cleared

## Decisions Made

- Migrated all 7 panels in a single Python script run for atomicity — all follow identical merge+reduce(sum) pattern so batch migration was safe
- TOU SQL (`CASE WHEN` clauses for Thai peak/off-peak rate classification) left completely unchanged — only the targets/transformations structure was modified
- `$A + $B` expression sums East inverter query (refId=A) + West inverter query (refId=B) for each financial metric

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None - no external service configuration required. The updated `solar-pv-monitor.json` can be imported directly into Grafana.

## Next Phase Readiness

- Phase 5 is fully complete — all calculation panels use server-side Expression targets, no client-side transformation arithmetic remains
- Phase 5.1 (Today data fix: change `now() - INTERVAL '24 hours'` to `date_trunc('day', now())`) is the next pending phase
- Dashboard is ready for Phase 5.1 execution

---
*Phase: 05-fix-transformations-to-expressions*
*Completed: 2026-04-01*
