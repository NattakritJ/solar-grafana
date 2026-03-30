# Requirements: Solar Monitoring Grafana Dashboard

**Defined:** 2026-03-30
**Core Value:** At a glance, the homeowner can see how much solar energy is being produced, how much the house is consuming, and how much money is being saved — down to the individual panel level.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### System Overview

- [x] **OVER-01**: User can see current total solar production power (kW) as a large prominent stat
- [x] **OVER-02**: User can see today's total energy production (kWh) as a stat
- [x] **OVER-03**: User can see lifetime total energy production (kWh/MWh) as a stat
- [x] **OVER-04**: User can see current grid import power (kW) from smart meter as a stat
- [x] **OVER-05**: User can see current house load (kW) calculated as solar production + grid import
- [x] **OVER-06**: User can see self-consumption ratio (%) showing how much load is covered by solar
- [x] **OVER-07**: User can see system status indicator (Normal/Fault/Offline) with color-coded value mapping
- [x] **OVER-08**: User can see power flow overview with stat panels arranged as Solar -> House <- Grid showing energy direction
- [x] **OVER-09**: User can see peak power achieved today (kW) as a stat

### Production Charts

- [x] **PROD-01**: User can see a production time-series graph showing power output over the selected time range
- [x] **PROD-02**: User can see per-inverter production breakdown (East vs West) as separate stats or grouped display
- [x] **PROD-03**: User can see production history as daily/weekly/monthly bar charts depending on selected time range
- [x] **PROD-04**: User can see East vs West production overlaid on the same time-series chart for orientation comparison
- [ ] **PROD-05**: User can see a calendar heatmap showing daily production patterns over weeks/months

### Module-Level Monitoring

- [x] **MODL-01**: User can see individual power output (W) for all 8 panels (PV1-PV4 on each inverter)
- [x] **MODL-02**: User can see individual voltage (V) and current (A) for all 8 panels
- [x] **MODL-03**: User can see today's and total production (kWh) for each individual panel
- [ ] **MODL-04**: User can see a roof layout visualization (Canvas panel) showing physical panel positions with color-coded production heatmap — East row: [PV3][PV4][PV1][PV2], West row: [PV3][PV4][PV1][PV2]

### Financial

- [ ] **FINC-01**: User can see today's estimated savings in THB based on solar production
- [ ] **FINC-02**: User can see TOU-aware savings calculations using peak rate (5.7982 THB/kWh, weekday 09-22) and off-peak rate (2.6369 THB/kWh, weekday 22-09, weekends/holidays all day)
- [ ] **FINC-03**: User can see cumulative savings for month-to-date and year-to-date periods
- [ ] **FINC-04**: User can see savings breakdown showing peak kWh vs off-peak kWh and their respective THB values

### Inverter Health

- [x] **HLTH-01**: User can see inverter temperature for both East and West micro inverters as gauges or stats
- [x] **HLTH-02**: User can see device state, alarm status, and fault status for both inverters with color-coded indicators
- [x] **HLTH-03**: User can see alarm/fault history as a table log showing when alarms or faults occurred
- [x] **HLTH-04**: Dashboard accounts for normal inverter shutdown at night (no PV input = offline is expected, not alarming)

### Grid Quality

- [x] **GRID-01**: User can see grid voltage (V) and frequency (Hz) from smart meter as stats
- [x] **GRID-02**: User can see smart meter detailed view including power factor, current, voltage, and frequency as time-series trends

### Anomaly & Event Log

- [x] **EVNT-01**: User can see a grid backfeed event log table showing timestamp, power value (W), and duration for all instances where smart meter power_sensor < 0
- [x] **EVNT-02**: User can see today's backfeed event count and max backfeed power (W) as summary stats
- [x] **EVNT-03**: User can see a unified event log combining grid backfeed events, inverter alarm/fault events, and device state changes in chronological order

### Dashboard Infrastructure

- [x] **INFR-01**: Dashboard is delivered as a Grafana JSON export file importable via Grafana UI
- [x] **INFR-02**: All queries use InfluxDB 3 Core SQL syntax (not Flux, not InfluxQL)
- [x] **INFR-03**: All queries reference datasource `influxdb-solar` with database `solar`
- [x] **INFR-04**: Dashboard works with Grafana 12.4.1 built-in panel types (no external plugins required)
- [x] **INFR-05**: Dashboard responds to Grafana's time range picker for all time-dependent panels
- [x] **INFR-06**: All panels use appropriate units (W, kW, kWh, V, A, Hz, THB, %, C)

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Enhanced Visualization

- **VIS-01**: Animated energy flow diagram (Sankey-style) showing proportional energy flow
- **VIS-02**: CO2 offset estimation based on Thailand grid emission factor

### Advanced Analytics

- **ANLYT-01**: System efficiency gauge comparing actual vs theoretical production
- **ANLYT-02**: Power losses tracking and trend analysis from inverter loss data

### Alerting

- **ALRT-01**: Grafana alerting rules for zero production during daylight, inverter faults, over-temperature, grid anomalies (separate from dashboard JSON)

## Out of Scope

| Feature | Reason |
|---------|--------|
| Battery management/SOC | No battery in system |
| Export revenue calculations | Zero-export system, no feed-in tariff |
| Weather forecast integration | External API dependency, not worth complexity for single-home dashboard |
| Production forecasting/prediction | Requires ML models, massive scope creep |
| Multi-site management | Single residential system |
| Mobile-specific layout | Grafana handles responsive natively |
| Inverter firmware management | Outside dashboard scope |
| Custom alerting in dashboard JSON | Grafana Alerting is a separate concern; document recommended alerts but don't embed |
| InfluxDB/Grafana server setup | Already running and configured |
| Data collection pipeline | Already feeding InfluxDB |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| OVER-01 | Phase 1 | Complete |
| OVER-02 | Phase 1 | Complete |
| OVER-03 | Phase 1 | Complete |
| OVER-04 | Phase 1 | Complete |
| OVER-05 | Phase 1 | Complete |
| OVER-06 | Phase 1 | Complete |
| OVER-07 | Phase 1 | Complete |
| OVER-08 | Phase 1 | Complete |
| OVER-09 | Phase 1 | Complete |
| PROD-01 | Phase 2 | Complete |
| PROD-02 | Phase 2 | Complete |
| PROD-03 | Phase 2 | Complete |
| PROD-04 | Phase 2 | Complete |
| PROD-05 | Phase 4 | Pending |
| MODL-01 | Phase 3 | Complete |
| MODL-02 | Phase 3 | Complete |
| MODL-03 | Phase 3 | Complete |
| MODL-04 | Phase 4 | Pending |
| FINC-01 | Phase 4 | Pending |
| FINC-02 | Phase 4 | Pending |
| FINC-03 | Phase 4 | Pending |
| FINC-04 | Phase 4 | Pending |
| HLTH-01 | Phase 3 | Complete |
| HLTH-02 | Phase 3 | Complete |
| HLTH-03 | Phase 3 | Complete |
| HLTH-04 | Phase 3 | Complete |
| GRID-01 | Phase 2 | Complete |
| GRID-02 | Phase 2 | Complete |
| EVNT-01 | Phase 3 | Complete |
| EVNT-02 | Phase 3 | Complete |
| EVNT-03 | Phase 3 | Complete |
| INFR-01 | Phase 1 | Complete |
| INFR-02 | Phase 1 | Complete |
| INFR-03 | Phase 1 | Complete |
| INFR-04 | Phase 1 | Complete |
| INFR-05 | Phase 1 | Complete |
| INFR-06 | Phase 1 | Complete |

**Coverage:**
- v1 requirements: 37 total
- Mapped to phases: 37
- Unmapped: 0

---
*Requirements defined: 2026-03-30*
*Last updated: 2026-03-30 after adding anomaly/event log requirements*
