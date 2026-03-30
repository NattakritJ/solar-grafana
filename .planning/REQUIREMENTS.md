# Requirements: Solar Monitoring Grafana Dashboard

**Defined:** 2026-03-30
**Core Value:** At a glance, the homeowner can see how much solar energy is being produced, how much the house is consuming, and how much money is being saved — down to the individual panel level.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### System Overview

- [ ] **OVER-01**: User can see current total solar production power (kW) as a large prominent stat
- [ ] **OVER-02**: User can see today's total energy production (kWh) as a stat
- [ ] **OVER-03**: User can see lifetime total energy production (kWh/MWh) as a stat
- [ ] **OVER-04**: User can see current grid import power (kW) from smart meter as a stat
- [ ] **OVER-05**: User can see current house load (kW) calculated as solar production + grid import
- [ ] **OVER-06**: User can see self-consumption ratio (%) showing how much load is covered by solar
- [ ] **OVER-07**: User can see system status indicator (Normal/Fault/Offline) with color-coded value mapping
- [ ] **OVER-08**: User can see power flow overview with stat panels arranged as Solar -> House <- Grid showing energy direction
- [ ] **OVER-09**: User can see peak power achieved today (kW) as a stat

### Production Charts

- [ ] **PROD-01**: User can see a production time-series graph showing power output over the selected time range
- [ ] **PROD-02**: User can see per-inverter production breakdown (East vs West) as separate stats or grouped display
- [ ] **PROD-03**: User can see production history as daily/weekly/monthly bar charts depending on selected time range
- [ ] **PROD-04**: User can see East vs West production overlaid on the same time-series chart for orientation comparison
- [ ] **PROD-05**: User can see a calendar heatmap showing daily production patterns over weeks/months

### Module-Level Monitoring

- [ ] **MODL-01**: User can see individual power output (W) for all 8 panels (PV1-PV4 on each inverter)
- [ ] **MODL-02**: User can see individual voltage (V) and current (A) for all 8 panels
- [ ] **MODL-03**: User can see today's and total production (kWh) for each individual panel
- [ ] **MODL-04**: User can see a roof layout visualization (Canvas panel) showing physical panel positions with color-coded production heatmap — East row: [PV3][PV4][PV1][PV2], West row: [PV3][PV4][PV1][PV2]

### Financial

- [ ] **FINC-01**: User can see today's estimated savings in THB based on solar production
- [ ] **FINC-02**: User can see TOU-aware savings calculations using peak rate (5.7982 THB/kWh, weekday 09-22) and off-peak rate (2.6369 THB/kWh, weekday 22-09, weekends/holidays all day)
- [ ] **FINC-03**: User can see cumulative savings for month-to-date and year-to-date periods
- [ ] **FINC-04**: User can see savings breakdown showing peak kWh vs off-peak kWh and their respective THB values

### Inverter Health

- [ ] **HLTH-01**: User can see inverter temperature for both East and West micro inverters as gauges or stats
- [ ] **HLTH-02**: User can see device state, alarm status, and fault status for both inverters with color-coded indicators
- [ ] **HLTH-03**: User can see alarm/fault history as a table log showing when alarms or faults occurred
- [ ] **HLTH-04**: Dashboard accounts for normal inverter shutdown at night (no PV input = offline is expected, not alarming)

### Grid Quality

- [ ] **GRID-01**: User can see grid voltage (V) and frequency (Hz) from smart meter as stats
- [ ] **GRID-02**: User can see smart meter detailed view including power factor, current, voltage, and frequency as time-series trends

### Dashboard Infrastructure

- [ ] **INFR-01**: Dashboard is delivered as a Grafana JSON export file importable via Grafana UI
- [ ] **INFR-02**: All queries use InfluxDB 3 Core SQL syntax (not Flux, not InfluxQL)
- [ ] **INFR-03**: All queries reference datasource `influxdb-solar` with database `solar`
- [ ] **INFR-04**: Dashboard works with Grafana 12.4.1 built-in panel types (no external plugins required)
- [ ] **INFR-05**: Dashboard responds to Grafana's time range picker for all time-dependent panels
- [ ] **INFR-06**: All panels use appropriate units (W, kW, kWh, V, A, Hz, THB, %, C)

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
| OVER-01 | — | Pending |
| OVER-02 | — | Pending |
| OVER-03 | — | Pending |
| OVER-04 | — | Pending |
| OVER-05 | — | Pending |
| OVER-06 | — | Pending |
| OVER-07 | — | Pending |
| OVER-08 | — | Pending |
| OVER-09 | — | Pending |
| PROD-01 | — | Pending |
| PROD-02 | — | Pending |
| PROD-03 | — | Pending |
| PROD-04 | — | Pending |
| PROD-05 | — | Pending |
| MODL-01 | — | Pending |
| MODL-02 | — | Pending |
| MODL-03 | — | Pending |
| MODL-04 | — | Pending |
| FINC-01 | — | Pending |
| FINC-02 | — | Pending |
| FINC-03 | — | Pending |
| FINC-04 | — | Pending |
| HLTH-01 | — | Pending |
| HLTH-02 | — | Pending |
| HLTH-03 | — | Pending |
| HLTH-04 | — | Pending |
| GRID-01 | — | Pending |
| GRID-02 | — | Pending |
| INFR-01 | — | Pending |
| INFR-02 | — | Pending |
| INFR-03 | — | Pending |
| INFR-04 | — | Pending |
| INFR-05 | — | Pending |
| INFR-06 | — | Pending |

**Coverage:**
- v1 requirements: 34 total
- Mapped to phases: 0
- Unmapped: 34

---
*Requirements defined: 2026-03-30*
*Last updated: 2026-03-30 after initial definition*
