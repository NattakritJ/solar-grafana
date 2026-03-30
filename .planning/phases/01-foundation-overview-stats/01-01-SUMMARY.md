---
phase: 01-foundation-overview-stats
plan: "01"
subsystem: infra
tags: [grafana, influxdb, sql, dashboard, json]

# Dependency graph
requires: []
provides:
  - Importable Grafana 12.4.1 dashboard JSON with __inputs datasource pattern
  - Dashboard skeleton with 7 collapsible row sections for all 4 phases
  - 3 overview stat panels: Solar Power (kW), Today's Energy (kWh), Lifetime Energy (kWh)
  - Established SQL query patterns: COALESCE null handling, double-quoted measurement names, UNION ALL for multi-source aggregation
affects:
  - 01-02 (next plan in Phase 1 — adds more overview panels to this JSON)
  - All subsequent phases (all phases add panels to solar-pv-monitor.json)

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "__inputs datasource pattern for portable Grafana dashboard import"
    - "UNION ALL subquery pattern for aggregating data from East + West Microinverter"
    - "COALESCE(field, 0) for nighttime NULL handling in all power fields"
    - "Double-quoted measurement names for InfluxDB 3 SQL table names with spaces"
    - "Stat panels with transparent=true for clean KPI bar layout"

key-files:
  created:
    - solar-pv-monitor.json
  modified: []

key-decisions:
  - "Used UNION ALL single-query approach instead of two-query + transformation for Solar Power panel (simpler, no Grafana transformation required)"
  - "COALESCE applied to all power fields to handle nighttime nulls as per PITFALLS.md Pitfall 9"
  - "No time-range macros ($__timeFrom/$__timeTo) in Phase 1 — deferred to Phase 2 to avoid macro substitution risk"
  - "All stat panels use transparent=true for clean KPI bar appearance"

patterns-established:
  - "Pattern 1: All measurement names double-quoted in SQL: \"East Microinverter\", \"West Microinverter\", \"Smart Meter\""
  - "Pattern 2: UNION ALL subquery pattern for aggregating East + West inverter data"
  - "Pattern 3: __inputs datasource reference — every panel uses { type: 'influxdb', uid: '${DS_INFLUXDB_SOLAR}' }"
  - "Pattern 4: Latest-value queries use WHERE time >= now() - INTERVAL '...' ORDER BY time DESC LIMIT 1"

requirements-completed: [INFR-01, INFR-02, INFR-03, INFR-04, INFR-05, INFR-06, OVER-01, OVER-02, OVER-03]

# Metrics
duration: 5min
completed: "2026-03-30"
---

# Phase 01 Plan 01: Foundation & Overview Stats — Dashboard Skeleton Summary

**Importable Grafana 12.4.1 dashboard JSON with __inputs datasource pattern, 7-section skeleton, and 3 overview stat panels (Solar Power kW, Today's Energy kWh, Lifetime Energy kWh) using InfluxDB 3 SQL**

## Performance

- **Duration:** ~5 min
- **Started:** 2026-03-30T03:59:51Z
- **Completed:** 2026-03-30T04:01:00Z
- **Tasks:** 2 completed
- **Files modified:** 1

## Accomplishments
- Created valid, importable `solar-pv-monitor.json` with `__inputs` pattern (Grafana prompts for datasource on import)
- Dashboard skeleton with all 7 collapsible row sections for all 4 phases (Overview expanded, Production through Event Log collapsed)
- 3 transparent stat panels: Solar Power (kW), Today's Energy (kWh), Lifetime Energy (kWh) — all with threshold color coding
- Established core SQL patterns: COALESCE for null handling, double-quoted measurement names, UNION ALL for multi-inverter aggregation

## Task Commits

Each task was committed atomically:

1. **Task 1: Create dashboard JSON skeleton with __inputs, metadata, and 7 section rows** - `66ce88c` (feat)
2. **Task 2: Add 3 overview stat panels — Solar Power, Today's Energy, Lifetime Energy** - `5f583b2` (feat)

**Plan metadata:** _(final docs commit — see below)_

## Files Created/Modified
- `solar-pv-monitor.json` — Complete Grafana dashboard JSON: __inputs pattern, schemaVersion 40, Asia/Bangkok timezone, 10s refresh, 7 row sections, 3 stat panels with InfluxDB 3 SQL queries

## Decisions Made
- Used UNION ALL single-query approach for Solar Power panel instead of two separate queries + Grafana merge/reduce transformations (simpler JSON, no transformation dependency)
- COALESCE applied to all power fields to handle nighttime NULLs per PITFALLS.md Pitfall 9
- No `$__timeFrom`/`$__timeTo` macros in Phase 1 queries — used `now() - INTERVAL '...'` pattern to avoid macro substitution risk (PITFALLS.md Pitfall 10, STATE.md blocker)
- All stat panels use `transparent: true` for clean unified KPI bar appearance
- Today's Energy uses `MAX(today_production_{1-4}_sensor)` within 24h window (daily counter fields that reset each day, so MAX = current day value)
- Lifetime Energy uses `MAX(total_production_{1-4}_sensor)` within 1h window (cumulative monotonic counter, so MAX = latest value)

## Deviations from Plan

None — plan executed exactly as written.

## Issues Encountered

None — all queries follow InfluxDB 3 Core SQL syntax as specified. JSON validated successfully with `node -e "JSON.parse(...)"`.

## User Setup Required

None - dashboard is self-contained. On import into Grafana, user will be prompted to map `DS_INFLUXDB_SOLAR` to their `influxdb-solar` datasource (this is by design via the `__inputs` pattern).

## Known Stubs

None — the 3 stat panels have complete SQL queries. No placeholder data or hardcoded mock values. Panels will show "No Data" if InfluxDB is unreachable, which is correct behavior (not a stub).

## Next Phase Readiness
- `solar-pv-monitor.json` is the single dashboard artifact — all subsequent plans add panels to this file
- Phase 1 Plan 02 adds remaining overview stat panels (Grid Import, House Load, Self-Consumption %, Peak Today, System Status, Power Flow visualization)
- The SQL query patterns established here (COALESCE, double-quoted names, UNION ALL) must be maintained in all subsequent queries
- Time-range macro validation (`$__timeFrom`/`$__timeTo`) is still needed — Phase 2 first use of time-series charts will validate this

---
*Phase: 01-foundation-overview-stats*
*Completed: 2026-03-30*
