# Solar Monitoring Grafana Dashboard

## What This Is

A comprehensive Grafana dashboard for monitoring a residential rooftop solar PV system. It visualizes real-time and historical data from two micro inverters (8 panels total) and a smart meter, all stored in InfluxDB 3 Core. The dashboard provides total production, per-inverter production, module-level monitoring with roof layout visualization, grid import/consumption tracking, and financial savings calculations.

## Core Value

At a glance, the homeowner can see how much solar energy is being produced right now, how much the house is consuming, and how much money is being saved — down to the individual panel level.

## Requirements

### Validated

- [x] Total solar production (real-time power + cumulative energy) — Validated in Phase 1: Foundation & Overview Stats
- [x] Grid import power (real-time from smart meter) — Validated in Phase 1: Foundation & Overview Stats
- [x] House load calculation (solar production + grid import) — Validated in Phase 1: Foundation & Overview Stats
- [x] Today/total energy production stats — Validated in Phase 1: Foundation & Overview Stats

### Active

- [ ] Per-inverter production breakdown (East vs West)
- [ ] Module-level monitoring (8 individual panels with power, voltage, current)
- [ ] Roof layout visualization showing panel positions with production heatmap
- [ ] Financial savings calculation using TOU rates
- [ ] Inverter health monitoring (temperature, alarms, faults, state)
- [ ] Grid quality monitoring (voltage, frequency, power factor)
- [ ] Historical trends (production over time, daily/weekly/monthly)
- [ ] All data from InfluxDB utilized in meaningful visualizations

### Out of Scope

- InfluxDB server setup/configuration — already running
- Grafana server setup/configuration — already running
- Data collection pipeline (Home Assistant/telegraf/etc.) — already feeding InfluxDB
- Battery storage monitoring — no battery in system
- Export energy tracking beyond what smart meter reports — zero export system
- Mobile app — Grafana's built-in responsive/mobile view is sufficient

## Context

### Infrastructure (Already Running)

- **InfluxDB 3 Core** at `192.168.2.10:8181`, bucket: `solar`
- **Grafana 12.4.1** with InfluxDB datasource named `influxdb-solar` already configured
- InfluxDB uses SQL query language (v3 Core), ~10-second reporting interval
- Data collection recently started (system is new)

### Solar System Hardware

- **Panels:** 8x Longi LR7-72HVDF-655M (655W each) = 5.24 kWp total capacity
- **Inverters:** 2x micro inverters, each rated 2250W with 4 MPPT inputs
  - "East Microinverter" — 4 panels on east-facing roof slope
  - "West Microinverter" — 4 panels on west-facing roof slope
- **Smart Meter:** Grid import/export metering at point of connection
- **Configuration:** Grid-tied, zero export (no feed-in tariff)

### Panel Physical Layout

Both East and West rows: 4 panels in a single row, portrait orientation.

Physical position (left to right when facing roof):
```
East Row:  [PV3] [PV4] [PV1] [PV2]
West Row:  [PV3] [PV4] [PV1] [PV2]
```

### InfluxDB Data Structure

**3 measurements** (tables):

| Measurement | Key Fields |
|---|---|
| `East Microinverter` | power_sensor, pv_power_sensor, pv{1-4}_power/voltage/current_sensor, grid_voltage/current/frequency_sensor, temperature_sensor, today_production_{1-4}_sensor, total_production_{1-4}_sensor, power_losses_sensor, device_state/alarm/fault_sensor |
| `West Microinverter` | Same structure as East |
| `Smart Meter` | power_sensor, voltage_sensor, current_sensor, frequency_sensor, power_factor_sensor, total_import_energy_sensor, total_export_energy_sensor |

- `device_name` and `device_type` are tag columns (Dictionary type)
- All numeric fields are Float64
- Timestamp column: `time` (nanosecond precision)

### Electricity Rates (TOU — Time of Use)

| Period | Hours | Rate (THB/kWh) |
|---|---|---|
| **Peak** | Weekday (Mon-Fri) 09:00-22:00 | 5.7982 |
| **Off-Peak** | Weekday 22:00-09:00, Weekend/Holiday all day | 2.6369 |

Currency: Thai Baht (THB). No feed-in tariff (zero export).

Solar production mostly occurs during peak hours (daytime), maximizing savings value.

## Constraints

- **Grafana Version**: 12.4.1 — use panel types and plugins compatible with this version
- **Query Language**: InfluxDB 3 Core uses SQL (not InfluxQL or Flux) — all queries must be SQL
- **Datasource**: Must reference `influxdb-solar` as the datasource name in dashboard JSON
- **Delivery Format**: Grafana JSON dashboard export file, importable via Grafana UI
- **Single Dashboard**: All monitoring on one comprehensive dashboard with logical sections
- **Zero Export**: System cannot export to grid — no export revenue calculations needed

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Single dashboard (not multiple) | User wants everything at a glance in one place | Confirmed — Phase 1 delivers single importable JSON |
| JSON export delivery | Simple import, no provisioning infrastructure needed | Confirmed — __inputs pattern works for datasource binding |
| SQL queries (InfluxDB v3) | InfluxDB 3 Core only supports SQL, not Flux/InfluxQL | Confirmed — all queries use valid InfluxDB 3 SQL |
| TOU-aware financial calculations | Peak solar production aligns with peak rates, maximizing tracked savings | — Pending |
| Roof layout visualization | Module-level visual showing physical panel positions with production heatmap | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-03-30 after Phase 1 completion — Foundation & Overview Stats*
