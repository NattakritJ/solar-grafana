# Roadmap: Solar Monitoring Grafana Dashboard

## Overview

This roadmap delivers a comprehensive Grafana dashboard JSON for monitoring a 5.24 kWp residential solar PV system. We start by validating InfluxDB 3 SQL connectivity and building the always-visible overview stats (the #1 risk gate), then layer on production charts and grid monitoring, followed by module-level per-panel detail and inverter health, and finally tackle the highest-complexity features: TOU financial calculations, Canvas roof layout heatmap, and visual polish. Each phase delivers a usable, importable dashboard — Phase 1 gives basic monitoring, and each subsequent phase adds a complete capability layer.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [x] **Phase 1: Foundation & Overview Stats** - Dashboard skeleton with validated SQL queries, header stats showing real-time power/energy/status, and power flow layout
- [x] **Phase 2: Production Charts & Grid Monitoring** - Time-series production graphs, East vs West comparison, daily/weekly bars, and grid quality views
- [x] **Phase 3: Module-Level Detail, Inverter Health & Event Log** - Per-panel power/voltage/current for all 8 panels, temperature gauges, alarm/fault history, grid backfeed event tracking
- [x] **Phase 4: Financial Savings, Canvas Layout & Polish** - TOU-aware savings calculations, roof layout heatmap visualization, calendar heatmap, unit/color consistency (completed 2026-03-30)

## Phase Details

### Phase 1: Foundation & Overview Stats
**Goal**: User can import the dashboard and see real-time solar production, grid import, house load, and system status at a glance
**Depends on**: Nothing (first phase)
**Requirements**: INFR-01, INFR-02, INFR-03, INFR-04, INFR-05, INFR-06, OVER-01, OVER-02, OVER-03, OVER-04, OVER-05, OVER-06, OVER-07, OVER-08, OVER-09
**Success Criteria** (what must be TRUE):
  1. Dashboard JSON can be imported into Grafana 12.4.1 via UI without errors, prompting for the `influxdb-solar` datasource
  2. User sees current total solar power (kW), today's energy (kWh), lifetime energy, grid import power, house load, self-consumption ratio, peak power today, and system status as stat panels in a header row
  3. User sees a power flow arrangement (Solar → House ← Grid) that communicates energy direction at a glance
  4. All panels respond to Grafana's time range picker and display correct units (kW, kWh, %)
  5. All queries use valid InfluxDB 3 SQL syntax (DATE_BIN, AVG, etc.) and return data from the live system
**Plans:** 2 plans
Plans:
- [x] 01-01-PLAN.md — Dashboard skeleton with __inputs, 7 section rows, and 3 overview stats (Solar Power, Today's Energy, Lifetime Energy)
- [x] 01-02-PLAN.md — Grid import, house load, self-consumption, peak power, system status stats, and power flow row
**UI hint**: yes

### Phase 2: Production Charts & Grid Monitoring
**Goal**: User can analyze solar production patterns over time, compare East vs West inverter performance, and monitor grid quality
**Depends on**: Phase 1
**Requirements**: PROD-01, PROD-02, PROD-03, PROD-04, GRID-01, GRID-02
**Success Criteria** (what must be TRUE):
  1. User can see a time-series graph of total solar production over the selected time range showing the characteristic daily bell curve
  2. User can see East and West inverter production overlaid on the same chart and as separate stat/bar gauge values for direct comparison
  3. User can see daily/weekly/monthly production as bar charts that adapt to the selected time range
  4. User can see grid voltage (V), frequency (Hz), power factor, and current as stats and time-series trends from the smart meter
**Plans**: 2 plans
Plans:
- [x] 02-01-PLAN.md — Solar production time-series (East/West overlay), inverter bar gauge, East/West pie chart
- [x] 02-02-PLAN.md — Daily energy bar chart, grid quality stats, grid parameter time-series
**UI hint**: yes

### Phase 3: Module-Level Detail, Inverter Health & Event Log
**Goal**: User can monitor individual panel performance, inverter health, and system anomalies (including grid backfeed events) to diagnose issues at the component level
**Depends on**: Phase 2
**Requirements**: MODL-01, MODL-02, MODL-03, HLTH-01, HLTH-02, HLTH-03, HLTH-04, EVNT-01, EVNT-02, EVNT-03
**Success Criteria** (what must be TRUE):
  1. User can see power output (W) for all 8 individual panels (PV1-PV4 on East and West inverters)
  2. User can see voltage (V) and current (A) for all 8 panels, plus today's and total production per panel
  3. User can see inverter temperature for both micro inverters as gauges or stats with appropriate thresholds
  4. User can see device state, alarm, and fault status with color-coded indicators, plus a historical alarm/fault log table *(post-v1: dedicated table panel removed; state visible via stat panels + state-timeline)*
  5. Dashboard displays inverter offline status at night as normal/expected (not as an error condition)
  6. User can see a grid backfeed event log showing timestamp, power, and duration for all negative smart meter readings, plus today's backfeed count and max backfeed power as summary stats
  7. User can see a unified event log combining grid backfeed, inverter alarms/faults, and state changes chronologically *(post-v1: unified log panel removed; row 700 renamed "Backfeed Log")*
**Plans:** 3 plans
Plans:
- [x] 03-01-PLAN.md — Module-level bar gauge (8-panel power) and detail table (power/voltage/current/today/total)
- [x] 03-02-PLAN.md — Inverter temperature gauges, device state/alarm/fault stats, state timeline
- [x] 03-03-PLAN.md — Backfeed summary stats, backfeed event log, alarm/fault history, unified event log *(post-v1: panels 36+37 removed, row 700 renamed "Backfeed Log")*
**UI hint**: yes

### Phase 4: Financial Savings, Canvas Layout & Polish
**Goal**: User can see financial savings with TOU-aware calculations, visualize panel performance on a roof layout, and experience a polished, consistent dashboard
**Depends on**: Phase 3
**Requirements**: FINC-01, FINC-02, FINC-03, FINC-04, MODL-04, PROD-05
**Success Criteria** (what must be TRUE):
  1. User can see today's estimated savings in THB, with correct TOU classification (peak 5.7982 THB/kWh weekday 09-22, off-peak 2.6369 THB/kWh otherwise) using Asia/Bangkok timezone
  2. User can see cumulative month-to-date and year-to-date savings, plus a breakdown of peak vs off-peak kWh and their respective THB values
  3. User can see a Canvas roof layout showing 8 panels in physical positions (East row: PV3-PV4-PV1-PV2, West row: PV3-PV4-PV1-PV2) with color-coded production heatmap
  4. User can see a calendar heatmap showing daily production patterns over weeks/months
  5. All panels across the entire dashboard use consistent color schemes, correct units, and have descriptive tooltips/descriptions
**Plans**: 3 plans
Plans:
- [x] 04-01-PLAN.md — TOU-aware financial savings stat panels (Today/MTD/YTD + Peak/Off-Peak breakdown)
- [x] 04-02-PLAN.md — Canvas roof layout heatmap and daily production heatmap
- [x] 04-03-PLAN.md — Dashboard-wide consistency polish and visual verification
**UI hint**: yes

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation & Overview Stats | 2/2 | Complete | 2026-03-30 |
| 2. Production Charts & Grid Monitoring | 2/2 | Complete | 2026-03-30 |
| 3. Module-Level Detail, Inverter Health & Event Log | 3/3 | Complete | 2026-03-30 |
| 4. Financial Savings, Canvas Layout & Polish | 3/3 | Complete   | 2026-03-30 |

## Post-v1 Dashboard Changes (2026-04-01)

Applied manually to `solar-pv-monitor.json` after the v1.0 milestone:

| Change | Panels Affected | Detail |
|--------|-----------------|--------|
| `schemaVersion` 40 → 42 | All | Grafana 12.4.1 compatibility; `pluginVersion: "12.4.1"` on all panels; threshold `value: null → 0` |
| Overview row redesign | 6, 7, 8 | 8 × w=3 → 5 × w=4 (y=1) + 2 × w=12 (y=5); downstream rows shifted +3 |
| House Load: server-side expressions | 5 | 3 queries + client transforms → Expression targets (`$East + $West + $Grid`) |
| Self-Consumption: server-side expressions | 6 | Client calculateField chain → Expression targets (Solar/Total/Self math) |
| House Load colour: orange | 5 | Thresholds changed from solar-green to orange (load convention) |
| `transparent: true` removed | 6, 8 | Panels now on dedicated second row, standard background |
| System Status `reduceOptions.fields` normalised | 8 | `'/.*/'` → `''` |
| Panels 36 & 37 removed | 36, 37 | Alarm/Fault History and unified Event Log tables removed |
| Row 700 renamed | 700 | "Event Log" → "Backfeed Log" (reflects remaining content: panels 33–35 only) |

### Phase 5: Fix all panel in dashboard that still use transformation to calcuate to use expression instead. For example, look at Self-Consumption or 🏠 House Load panel that use Expression.

**Goal:** Migrate all 15 calculation panels from client-side `merge + reduce(sum)` / `calculateField` transformations to Grafana server-side Expression targets (`type: math`) so arithmetic runs as `$A + $B` expressions rather than transformation pipelines
**Requirements**: OVER-03, OVER-05, OVER-06, FINC-01, FINC-02, FINC-03, FINC-04
**Depends on:** Phase 4
**Plans:** 2/2 plans complete

Plans:
- [x] 05-01-PLAN.md — Clean up panels 5/6/10 (remove leftover disabled transforms) and migrate overview/power-flow panels 1, 2, 3, 7, 9 to Expression targets
- [x] 05-02-PLAN.md — Migrate financial savings panels 38-44 to Expression targets + human verification checkpoint

### Phase 05.1: Fix how to get "Today" data. Currently, it use WHERE time >= now() - INTERVAL '24 hours' which meaning last 24 hours not "Today". "Today" should mean from the beginning of the current day at 00:00 to now. (INSERTED)

**Goal:** Fix all 18 SQL queries that use `now() - INTERVAL '24 hours'` / `now() - INTERVAL '1 day'` as a "today" boundary — replace with `date_trunc('day', now() AT TIME ZONE 'Asia/Bangkok') AT TIME ZONE 'UTC'` so "today" means the Bangkok calendar day, not a rolling 24-hour window.
**Requirements**: OVER-02, OVER-09, PROD-02, FINC-01, FINC-02, EVNT-02
**Depends on:** Phase 5
**Plans:** 1/1 plans complete

Plans:
- [x] 05.1-01-PLAN.md — Replace all 18 rolling-window boundaries with Bangkok calendar-day expression (2 global replaceAll edits)

### Phase 6: Financial Savings rework: use fixed rate (3.5 THB/kWh) instead of TOU rate

**Goal:** Replace the complex TOU peak/off-peak rate calculation with a single flat rate of 3.5 THB/kWh for all savings panels, and remove the now-irrelevant TOU breakdown panels (Peak kWh, Peak THB, Off-Peak kWh, Off-Peak THB)
**Requirements**: FINC-01, FINC-02, FINC-03
**Depends on:** Phase 5.1
**Plans:** 1/1 plans complete

Plans:
- [x] 06-01-PLAN.md — Replace TOU SQL with flat-rate 3.5 THB/kWh in panels 38/39/40, remove panels 41-44 + human verification

### Phase 7: Fix stale data when inverter goes offline at sunset

**Goal:** Add a 2-minute recency window (`WHERE time >= now() - INTERVAL '2 minutes'`) to all 53 latest-value queries across 23 panels so that when inverters go offline at sunset, stat/bargauge/canvas panels show "no data" instead of freezing at the last active reading
**Requirements**: OVER-01, OVER-02, OVER-04, OVER-05, PROD-01, MODL-01, MODL-02, HLTH-01, HLTH-02, HLTH-03, GRID-01
**Depends on:** Phase 6
**Plans:** 1 plan

Plans:
- [x] 07-01-PLAN.md — Add WHERE time >= now() - INTERVAL '2 minutes' to all 53 latest-value queries (6 surgical replaceAll edits) + human verification

### Phase 8: Update house load measurement to use CT meters on both consumer units, add per-floor power distribution panels, and add data reconciliation panel comparing calculated (grid + solar) vs actual CT measurement

**Goal:** Replace derived House Load calculation with CT direct measurement, surface per-floor power distribution, and add reconciliation panels so the homeowner can compare CT-measured vs calculated load
**Requirements**: OVER-03, OVER-05
**Depends on:** Phase 7
**Plans:** 1 plan

Plans:
- [x] 08-01-PLAN.md — Replace House Load targets (CT Floor1+Floor2), insert Power Distribution row 900 with 4 stat panels, shift downstream, add Floor 1/2 series to Power Profile + human verification

### Phase 9: Update panels to use new CT meter data for grid energy import (Current, Voltage, Real Power) instead of smart meter

**Goal:** Migrate 12 panels from the Smart Meter table to the new `grid` CT meter table for grid power, voltage, and current data, leaving frequency and power_factor panels on Smart Meter since those fields don't exist in the CT table
**Requirements**: GRID-01, GRID-02, OVER-04, OVER-05, OVER-06, EVNT-01, EVNT-02
**Depends on:** Phase 8
**Plans:** 1 plan

Plans:
- [ ] 09-01-PLAN.md — Apply 10 surgical SQL replacements across 12 panels to switch from Smart Meter to CT grid table + human verification
