# Roadmap: Solar Monitoring Grafana Dashboard

## Overview

This roadmap delivers a comprehensive Grafana dashboard JSON for monitoring a 5.24 kWp residential solar PV system. We start by validating InfluxDB 3 SQL connectivity and building the always-visible overview stats (the #1 risk gate), then layer on production charts and grid monitoring, followed by module-level per-panel detail and inverter health, and finally tackle the highest-complexity features: TOU financial calculations, Canvas roof layout heatmap, and visual polish. Each phase delivers a usable, importable dashboard — Phase 1 gives basic monitoring, and each subsequent phase adds a complete capability layer.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Foundation & Overview Stats** - Dashboard skeleton with validated SQL queries, header stats showing real-time power/energy/status, and power flow layout
- [ ] **Phase 2: Production Charts & Grid Monitoring** - Time-series production graphs, East vs West comparison, daily/weekly bars, and grid quality views
- [ ] **Phase 3: Module-Level Detail, Inverter Health & Event Log** - Per-panel power/voltage/current for all 8 panels, temperature gauges, alarm/fault history, grid backfeed event tracking
- [ ] **Phase 4: Financial Savings, Canvas Layout & Polish** - TOU-aware savings calculations, roof layout heatmap visualization, calendar heatmap, unit/color consistency

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
**Plans**: TBD
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
**Plans**: TBD
**UI hint**: yes

### Phase 3: Module-Level Detail, Inverter Health & Event Log
**Goal**: User can monitor individual panel performance, inverter health, and system anomalies (including grid backfeed events) to diagnose issues at the component level
**Depends on**: Phase 2
**Requirements**: MODL-01, MODL-02, MODL-03, HLTH-01, HLTH-02, HLTH-03, HLTH-04, EVNT-01, EVNT-02, EVNT-03
**Success Criteria** (what must be TRUE):
  1. User can see power output (W) for all 8 individual panels (PV1-PV4 on East and West inverters)
  2. User can see voltage (V) and current (A) for all 8 panels, plus today's and total production per panel
  3. User can see inverter temperature for both micro inverters as gauges or stats with appropriate thresholds
  4. User can see device state, alarm, and fault status with color-coded indicators, plus a historical alarm/fault log table
  5. Dashboard displays inverter offline status at night as normal/expected (not as an error condition)
  6. User can see a grid backfeed event log showing timestamp, power, and duration for all negative smart meter readings, plus today's backfeed count and max backfeed power as summary stats
  7. User can see a unified event log combining grid backfeed, inverter alarms/faults, and state changes chronologically
**Plans**: TBD
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
**Plans**: TBD
**UI hint**: yes

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation & Overview Stats | 0/TBD | Not started | - |
| 2. Production Charts & Grid Monitoring | 0/TBD | Not started | - |
| 3. Module-Level Detail, Inverter Health & Event Log | 0/TBD | Not started | - |
| 4. Financial Savings, Canvas Layout & Polish | 0/TBD | Not started | - |
