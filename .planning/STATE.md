---
gsd_state_version: 1.0
milestone: v1.0
milestone_name: milestone
status: verifying
stopped_at: Phase 11-01 Task 1 complete, awaiting human-verify checkpoint (Task 2)
last_updated: "2026-04-09T00:39:15.299Z"
last_activity: 2026-04-09
progress:
  total_phases: 12
  completed_phases: 12
  total_plans: 19
  completed_plans: 19
  percent: 100
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-03-30)

**Core value:** At a glance, the homeowner can see how much solar energy is being produced, how much the house is consuming, and how much money is being saved — down to the individual panel level.
**Current focus:** Phase 11 — change-table-for-solar-ac-power-output

## Current Position

Phase: 11
Plan: Not started
Status: Phase complete — ready for verification
Last activity: 2026-04-09

Progress: [██████████] 100%

## Performance Metrics

**Velocity:**

- Total plans completed: 0
- Average duration: —
- Total execution time: 0 hours

**By Phase:**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| - | - | - | - |

**Recent Trend:**

- Last 5 plans: —
- Trend: —

*Updated after each plan completion*
| Phase 01-foundation-overview-stats P01 | 5 | 2 tasks | 1 files |
| Phase 01-foundation-overview-stats P02 | 3min | 2 tasks | 1 files |
| Phase 02-production-charts-grid P01 | — | 5 tasks | 1 files |
| Phase 02-production-charts-grid P02 | — | 5 tasks | 1 files |
| Phase 03-module-level P01 | 3min | 2 tasks | 1 files |
| Phase 03-module-level P02 | 3min | 2 tasks | 1 files |
| Phase 03-module-level P03 | 3min | 2 tasks | 1 files |
| Phase 04 P02 | 3min | 2 tasks | 1 files |
| Phase 04 P01 | 19min | 2 tasks | 1 files |
| Phase 04 P03 | 3min | 1 tasks | 1 files |
| Phase 05 P01 | 2min | 2 tasks | 1 files |
| Phase 05 P02 | 5min | 2 tasks | 1 files |
| Phase 05.1 P01 | 1 | 1 tasks | 1 files |
| Phase 06 P01 | 2 | 1 tasks | 1 files |
| Phase 06-financial-savings-rework-use-fixed-rate-3-5-thb-kwh-instead-of-tou-rate P01 | 22min | 2 tasks | 1 files |
| Phase 08 P01 | 25min | 3 tasks | 1 files |
| Phase 11 P01 | 3min | 1 tasks | 1 files |

## Accumulated Context

### Decisions

Decisions are logged in PROJECT.md Key Decisions table.
Recent decisions affecting current work:

- [Roadmap]: 4-phase coarse structure — Foundation → Production/Grid → Module/Health → Financial/Canvas/Polish
- [Roadmap]: MODL-04 (Canvas roof layout) deferred to Phase 4 with Financial due to high complexity
- [Roadmap]: PROD-05 (calendar heatmap) grouped with Phase 4 as premium visualization feature
- [Phase 01-foundation-overview-stats]: ~~UNION ALL single-query~~ REPLACED: Multi-query + Grafana transformations (merge/reduce/calculateField) — InfluxDB 3 Core does NOT support SELECT FROM (subquery with UNION ALL)
- [Phase 01-foundation-overview-stats]: Phase 1 uses now()-INTERVAL pattern (no time-range macros) to avoid macro substitution validation risk
- [Phase 01-foundation-overview-stats]: __inputs pattern for DS_INFLUXDB_SOLAR — all panels use ${DS_INFLUXDB_SOLAR} uid for portable import
- [Phase 01-foundation-overview-stats]: ~~Cross-join subquery~~ REPLACED: Multi-query + transformations for multi-measurement calculations (House Load, Self-Consumption)
- [Phase 01-foundation-overview-stats]: Self-Consumption outputs 0-1 range with percentunit (not 0-100 with percent)
- [Phase 01-foundation-overview-stats]: Power flow panels use colorMode=background + graphMode=area to distinguish from KPI row
- [Bug Fix]: InfluxDB 3 Core SQL limitation — no subquery-wrapped UNION ALL. All multi-measurement panels must use separate Grafana queries + transformations.
- [Phase 02]: Time-series panels use $__timeFrom/$__timeTo macros for time-range awareness — needs runtime validation
- [Phase 02]: Multi-measurement time-series (production chart) use separate queries, Grafana overlays natively — no transformations needed
- [Phase 02]: Daily energy uses date_trunc('day') + MAX(today_production) counters from each inverter
- [Phase 02]: Grid stats use absolute thresholds for voltage (215-240V green), frequency (49.5-50.5Hz green), PF (0.9+ green)
  - [Phase 02]: Grid time-series use dual Y-axes (voltage+frequency, PF+current) for different-unit series
  - [Phase 03]: Per-panel queries use 8 separate queries + merge transformation (one per PV channel per inverter)
  - [Phase 03]: Production color scale: #1a1a2e → #614a19 → #c7a035 → #73BF69 → #1a7c11 (0W to 600W+)
  - [Phase 03]: HLTH-04: nighttime offline = dark-gray (not red) in all state panels and state-timeline
  - [Phase 03]: Device state mapping: 0=Offline/gray, 1=Standby/yellow, 2=Producing/green, 3=Fault/red
  - [Phase 03]: Temperature gauge thresholds: green <40°C, yellow 40-60°C, red >60°C
  - [Phase 03]: Backfeed detection: power_sensor < 0 from Smart Meter, displayed as ABS value
  - [Phase 03]: Unified event log: 3 separate queries + merge + sortBy transformations (no UNION ALL)
  - [Phase 03]: Event color coding: backfeed=purple, alarm=orange, fault=red, state change=blue
- [Phase 04]: Canvas uses string-type rectangle elements with absolute positioning for roof layout visualization
- [Phase 04]: Heatmap uses hourly DATE_BIN bucketing with Grafana calculate:true for daily grouping
- [Phase 04]: Hourly bucket TOU approach: AVG(power)/1000 per DATE_BIN hour for kWh estimation, classified by DOW+HOUR+holidays
- [Phase 04]: Thai public holidays encoded as month/day extraction in SQL WHERE clauses (15 dates for 2026)
- [Phase 04]: Financial panel Y positions adjusted from plan (y:64→y:82) due to Canvas+Heatmap panels inserted by parallel 04-02 execution
  - [Phase 04]: Production threshold colors normalized to consistent 5-step scale across all production panels for dashboard-wide consistency
- [Phase 05]: Migrated 8 stat panels from client-side merge+reduce/calculateField transformation chains to server-side Grafana Expression targets (type=math); raw InfluxDB query targets set to hide:true
- [Phase 05]: [Phase 05-02]: Migrated 7 financial savings panels (38-44) from merge+reduce transforms to Grafana Expression targets ( + ); Phase 5 migration complete — all 15 calculation panels now use server-side Expression arithmetic
- [Phase 05.1]: Calendar-day boundary for Bangkok timezone: date_trunc('day', now() AT TIME ZONE 'Asia/Bangkok') AT TIME ZONE 'UTC' replaces rolling 24h/1day windows in all 18 today-scoped queries
- [Phase 06]: Replaced TOU CASE WHEN SQL (5.7982 peak / 2.6369 off-peak + 15 Thai holiday exceptions) with SUM(hourly_kwh * 3.5) flat rate for savings panels 38/39/40 — homeowner's actual tariff is flat 3.5 THB/kWh; removed 4 TOU breakdown panels 41-44
- [Phase 06]: Flat rate 3.5 THB/kWh replaces TOU CASE WHEN SQL — homeowner's actual tariff is flat; TOU breakdown panels 41-44 removed; Bangkok timezone applied to all three savings boundaries
- [Phase 11]: Migrated 8 solar AC panels from inverter to CT tables (east/west_microinverter_power); 10s recency window for all CT snapshot panels; 5s dashboard refresh

### Post-v1 Edits Applied (2026-04-01)

The following decisions were recorded after the v1.0 milestone to reflect manual edits applied directly to `solar-pv-monitor.json`:

  - [Post-v1]: schemaVersion bumped 40 → 42 for Grafana 12.4.1; all panels updated to `pluginVersion: "12.4.1"` with full options schema; threshold step base `value: null` → `value: 0` normalized across all panels
  - [Post-v1]: Overview row redesigned — 8 × w=3 panels collapsed to 5 × w=4 panels at y=1, plus Self-Consumption (panel 6, w=12) and System Status (panel 8, w=12) on a new second row at y=5; all downstream rows shifted down 3 units
  - [Post-v1]: `transparent: true` removed from panels 6 (Self-Consumption) and 8 (System Status) as they now sit on their own dedicated row
  - [Post-v1]: House Load (panel 5) migrated from 3 InfluxDB queries + client-side merge/calculateField transforms to server-side Grafana Expression targets — `$East + $West + $Grid` (type=math); simpler and more reliable
  - [Post-v1]: Self-Consumption (panel 6) migrated to server-side Grafana Expressions: `$Solar = $East + $West`, `$Total = $Solar + $Grid`, `$Self = $Solar / $Total`
  - [Post-v1]: System Status (panel 8) `reduceOptions.fields` normalised from regex `'/.*/'` to `''` (empty string)
  - [Post-v1]: House Load colour thresholds changed from solar-green scale to orange — orange is the correct colour convention for load/consumption panels (per D-14)
  - [Post-v1]: Panels 36 ("Alarm / Fault History") and 37 ("Event Log" unified log) removed from dashboard; Row 700 renamed "Event Log" → "Backfeed Log"

### Post-v1 Edits Applied (2026-04-02)

The following decisions were recorded after the Phase 07 completion to reflect manual edits applied directly to `solar-pv-monitor.json` (dashboard version bumped 20 → 27):

  - [Post-v1 2026-04-02]: System Status (panel 8) display options reworked — `colorMode: value → none` (no per-value tinting), `justifyMode: auto → center`, `textMode: value → value_and_name` (shows inverter name alongside state), `wideLayout: true → false` (compact layout)
  - [Post-v1 2026-04-02]: `transparent: true` removed from three Power Flow row panels — ☀ Solar → House, 🏠 House Load, ⚡ Grid → House — these panels now render with standard panel background
  - [Post-v1 2026-04-02]: JSON cosmetic reformatting — 31 single-element arrays collapsed to inline style (`["lastNotNull"]` etc.); no functional change

### Roadmap Evolution

- Phase 5 added: Fix all panel in dashboard that still use transformation to calcuate to use expression instead. For example, look at Self-Consumption or 🏠 House Load panel that use Expression.
- Phase 5.1 inserted after Phase 5: Fix how to get "Today" data. Currently, it use WHERE time >= now() - INTERVAL '24 hours' which meaning last 24 hours not "Today". "Today" should mean from the beginning of the current day at 00:00 to now. (URGENT)
- Phase 6 added: Financial Savings rework: use fixed rate (3.5 THB/kWh) instead of TOU rate
- Phase 7 added: Fix stale data when inverter goes offline at sunset — most panels use ORDER BY time DESC LIMIT 1 which freezes at the last value when the inverter turns off at night; need time-bounded queries that return null/no data outside active hours
- Phase 8 added: Update house load measurement to use CT meters on both consumer units, add per-floor power distribution panels, and add data reconciliation panel comparing calculated (grid + solar) vs actual CT measurement
- Phase 9 added: Update panels to use new CT meter data for grid energy import (Current, Voltage, Real Power) instead of smart meter
- Phase 10 added: After using dashboard for a while, I found a major problem. Each device have different log time and almost all query use DESC LIMIT 1 so data coming to dashboard are not in the same timeframe. how can we tackle this problem?
- Phase 11 added: Change table for solar AC power output

### Pending Todos

None yet.

### Quick Tasks Completed

| # | Description | Date | Commit | Directory |
|---|-------------|------|--------|-----------|
| 260402-x4d | add README.md | 2026-04-02 | 681d76f | [260402-x4d-add-readme-md](./quick/260402-x4d-add-readme-md/) |
| 260409-b4w | remove COALESCE from 6 always-sending tables (21 queries) | 2026-04-09 | fa67d80 | [260409-b4w-remove-coalesce-from-sql-queries-for-tab](./quick/260409-b4w-remove-coalesce-from-sql-queries-for-tab/) |

### Blockers/Concerns

- [RESOLVED] ~~[Phase 1]: InfluxDB 3 SQL macro syntax — addressed by using now()-INTERVAL instead~~
- [RESOLVED] ~~[Phase 1]: UNION ALL subquery not supported — fixed with multi-query + transformations~~
- [RESOLVED] ~~[Phase 2+]: Multi-measurement time-series queries need same multi-query pattern (no UNION ALL)~~
- [Phase 2]: $__timeFrom / $__timeTo macro syntax needs runtime validation for time-range-aware queries
- [Phase 4]: Canvas panel JSON may require building in Grafana UI first then extracting JSON
- [Phase 4]: AT TIME ZONE 'Asia/Bangkok' interaction with Grafana macros needs validation
- [Phase 4]: Thai public holidays in TOU classification — business rules question

## Session Continuity

Last session: 2026-04-09T00:23:44.982Z
Stopped at: Phase 11-01 Task 1 complete, awaiting human-verify checkpoint (Task 2)
Resume file: None
