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
- [x] Per-inverter production breakdown (East vs West) — Validated in Phase 2: Production & Per-Inverter
- [x] Historical trends (production over time, daily/weekly/monthly) — Validated in Phase 2: Production & Per-Inverter
- [x] Module-level monitoring (8 individual panels with power, voltage, current) — Validated in Phase 3: Module Level, Grid & Inverter Health
- [x] Inverter health monitoring (temperature, alarms, faults, state) — Validated in Phase 3: Module Level, Grid & Inverter Health
- [x] Grid quality monitoring (voltage, frequency, power factor) — Validated in Phase 3: Module Level, Grid & Inverter Health
- [x] All data from InfluxDB utilized in meaningful visualizations — Validated in Phase 3: Module Level, Grid & Inverter Health
- [x] Financial savings calculation using TOU rates — Validated in Phase 4: Financial Savings, Canvas Layout & Polish
- [x] Roof layout visualization showing panel positions with production heatmap — Validated in Phase 4: Financial Savings, Canvas Layout & Polish

### Active

(None — all v1.0 requirements validated)

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

### Electricity Rate

| Rate | THB/kWh |
|------|---------|
| **Flat rate** | 3.5 |

Currency: Thai Baht (THB). No feed-in tariff (zero export).

The homeowner's actual tariff is a flat 3.5 THB/kWh — not TOU. The dashboard previously used complex TOU peak/off-peak rates (5.7982/2.6369) which were incorrect and have been replaced.

## Constraints

- **Grafana Version**: 12.4.1 — use panel types and plugins compatible with this version
- **Query Language**: InfluxDB 3 Core uses SQL (not InfluxQL or Flux) — all queries must be SQL
- **Datasource**: Must reference `influxdb-solar` as the datasource name in dashboard JSON
- **Delivery Format**: Grafana JSON dashboard export file, importable via Grafana UI
- **Single Dashboard**: All monitoring on one comprehensive dashboard with logical sections
- **Zero Export**: System cannot export to grid — no export revenue calculations needed
- **Schema Version**: `schemaVersion: 42` (bumped from 40 post-v1 for Grafana 12.4.1 full compatibility)

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Single dashboard (not multiple) | User wants everything at a glance in one place | Confirmed — Phase 1 delivers single importable JSON |
| JSON export delivery | Simple import, no provisioning infrastructure needed | Confirmed — __inputs pattern works for datasource binding |
| SQL queries (InfluxDB v3) | InfluxDB 3 Core only supports SQL, not Flux/InfluxQL | Confirmed — all queries use valid InfluxDB 3 SQL |
| TOU-aware financial calculations | Peak solar production aligns with peak rates, maximizing tracked savings | Confirmed — 7 stat panels with hourly-bucketed TOU SQL, Thai holiday handling |
| Roof layout visualization | Module-level visual showing physical panel positions with production heatmap | Confirmed — Canvas panel with 8 data-bound rectangles in physical arrangement |
| [Post-v1] Overview row redesign (2026-04-01) | Improve scannability: wider panels for primary KPIs, larger display area for Self-Consumption and System Status | 5×w=4 top row + two w=12 panels on second row; `transparent:true` removed from panels 6 & 8 |
| [Post-v1] Server-side Expressions for House Load & Self-Consumption (2026-04-01) | Grafana Expression targets (type=math) are simpler, less brittle than multi-step client-side transformation chains | Panels 5 and 6 use `$A + $B + $C` style math targets; no more merge/calculateField transformations |
| [Phase 5] Full Expression migration for all 15 calculation panels (2026-04-01) | Eliminated all remaining merge+reduce/calculateField transformation chains from calculation panels — single architectural pattern throughout dashboard | Panels 1, 2, 3, 7, 9 (overview), 5, 6, 10 (load/consumption), 38–44 (financial savings) all use Expression targets; display-only merge panels (13, 14, 15, 22, 23, 45) preserved |
| [Post-v1] Remove alarm/fault history & unified event log panels (2026-04-01) | Panels 36+37 duplicated info already visible in health stats; added noise without actionable value | Removed panels 36 & 37; Row 700 renamed to "Backfeed Log" (panels 33–35 only) |
| [Post-v1] Bump schemaVersion 40 → 42 (2026-04-01) | Required for Grafana 12.4.1 full compatibility; triggers updated panel options schema and normalised thresholds | `schemaVersion: 42`, `pluginVersion: "12.4.1"`, threshold base `value: null → 0` on all panels |
| [Phase 5.1] Calendar-day boundary for all "today" queries (2026-04-01) | `now() - INTERVAL '24 hours'` was a rolling window, not the Bangkok calendar day — "today" should mean since 00:00 Bangkok time | All 18 rawSql "today" boundaries replaced with `date_trunc('day', now() AT TIME ZONE 'Asia/Bangkok') AT TIME ZONE 'UTC'`; real-time patterns unchanged |
| [Phase 6] Flat-rate savings calculation replaces TOU (2026-04-01) | Homeowner's actual tariff is flat 3.5 THB/kWh — not TOU. The complex CASE WHEN SQL with two rates + 15 Thai public holiday exceptions was producing incorrect savings figures | Panels 38/39/40 use `SUM(hourly_kwh * 3.5)`; panels 41-44 (Peak/Off-Peak breakdown) removed; all three boundaries use Bangkok timezone anchor |
  | [Phase 7] 2-minute recency window for stale data fix (2026-04-01) | `ORDER BY time DESC LIMIT 1` with no time filter returns hours-old data after sunset — panels freeze at last active value | All 53 latest-value queries across 23 panels now include `WHERE time >= now() - INTERVAL '2 minutes'`; Panel 8 preserved with 1-minute filter |
| [Phase 8] CT direct measurement for House Load (2026-04-02) | Hardware CT meters on both consumer units (Floor 1, Floor 2) give a more accurate house load reading than the derived solar+grid formula | Panels 5 and 10 use `235_floor_1` + `235_floor_2` CT queries; new "Power Distribution" row (900) adds Floor 1, Floor 2, CT Load, and Calculated Load panels for per-floor visibility and reconciliation; Panel 801 Power Profile adds Floor 1/2 time-series |
  | [Phase 9] CT grid meter replaces Smart Meter for grid power/voltage/current (2026-04-04) | Dedicated CT sensor at grid connection point provides more accurate real-time measurement; signed power convention (negative = backfeed) enables proper backfeed visualization | 12 panels migrated from `Smart Meter` to `grid` table; Panel 801 Grid series removes GREATEST() clip; Panels 17/18 (frequency/PF) intentionally preserved on Smart Meter |
  | [Phase 10] AVG windowed aggregates replace all point-in-time queries (2026-04-04) | Each device logs at a different interval (inverters ~10s, CT meters ~2s) — `selector_last`/`LIMIT 1` returned values from different instants, causing cross-device expressions to combine non-contemporaneous readings | 62 `selector_last` → `AVG`, 16 production counters → `MAX`, 6 `ORDER BY DESC LIMIT 1` → `AVG`; 8 categorical `selector_last` (device state/alarm/fault) preserved |

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
*Last updated: 2026-04-04 — Phase 10 complete: all 84 point-in-time SQL queries replaced with AVG/MAX windowed aggregates; cross-device temporal misalignment resolved*
