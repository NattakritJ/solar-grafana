# Solar PV Monitoring Dashboard

Grafana dashboard for a residential 8-panel rooftop solar system with per-inverter, module-level, and financial tracking.

> _Screenshot: coming soon_

---

## Overview

This is a Grafana JSON dashboard that monitors a residential rooftop solar PV system in real time. It visualises data from two micro inverters (East and West, 4 panels each) and a Smart Meter — all stored in InfluxDB 3 Core. At a glance, the homeowner can see how much solar energy is being produced, how much the house is consuming (measured directly by CT meters on the monitored house circuits), and how much money is being saved — down to the individual panel level.

## Features

- **Total and per-inverter production** — real-time power (W) and cumulative energy (kWh) for East and West inverters
- **House load monitoring** — direct CT measurement on Floor 1, Floor 2, and 236 Floor 1 consumer units, plus per-floor power distribution panels
- **Grid import monitoring** — voltage, frequency, power factor, and current from Smart Meter
- **Module-level monitoring** — 8 individual panels showing power, voltage, and current per channel
- **Roof layout heatmap** — Canvas panel with physical panel positions colour-coded by production (0 W dark → 600 W+ bright green)
- **Financial savings** — daily, this-month, and all-time savings at the flat 3.5 THB/kWh tariff
- **Inverter health and state history** — temperature, alarm/fault status, and state timeline (offline / standby / producing / fault)
- **Data reconciliation** — comparison of CT-measured house load vs. the derived formula (grid import + solar production)

### Power Profile measurement model

The `Power Profile` chart now keeps each line on a single measurement model:

- **Production** — summed from the two inverter production queries
- **Grid** — direct from the `grid` measurement
- **Load** — direct CT total, calculated in Grafana as `Floor 1 + Floor 2 + 236 Floor 1`

The derived formula `production + grid import` still exists, but only in the dedicated **Calculated Load** stat panel where it is explicitly presented as a reconciliation value rather than the primary load line.
- **Backfeed log** — detects and logs any brief backfeed events from the zero-export Smart Meter

## Prerequisites

| Requirement | Detail |
|---|---|
| Grafana | 12.4.1 (OSS or Enterprise) — uses built-in panel types only, no extra plugins |
| InfluxDB | 3 Core with SQL query support |
| Datasource | A datasource named `influxdb-solar` configured in Grafana, pointing to your InfluxDB instance |
| Data pipeline | Data already flowing into the `solar` InfluxDB bucket with the three measurements described below |

## Installation / Import

1. In Grafana, go to **Dashboards → Import** (or **Dashboards → New → Import**).
2. Click **Upload dashboard JSON file** and select `solar-pv-monitor.json` (or paste its contents into the text field).
3. When prompted to select a datasource for the `influxdb-solar` input, choose your configured InfluxDB datasource.
4. Click **Import**.

The dashboard will be available immediately. Panels that rely on live data will populate once InfluxDB starts returning rows.

---

<details>
<summary><strong>Hardware Reference</strong></summary>

### Solar System

| Component | Detail |
|---|---|
| Panels | 8 × Longi LR7-72HVDF-655M, 655 W each = **5.24 kWp** total |
| East Microinverter | 2250 W rated, 4 MPPT inputs — 4 east-facing panels |
| West Microinverter | 2250 W rated, 4 MPPT inputs — 4 west-facing panels |
| Smart Meter | Grid connection metering (zero-export configuration, no feed-in tariff) |
| CT meters | Direct load measurements for `235_floor_1`, `235_floor_2`, and `236_floor_1` |

### Panel Physical Layout

Both rows are portrait orientation, left to right when facing the roof:

```
East Row:  [PV3] [PV4] [PV1] [PV2]
West Row:  [PV3] [PV4] [PV1] [PV2]
```

### Electricity Tariff

Flat rate: **3.5 THB / kWh** (no peak/off-peak TOU, no feed-in tariff).

</details>

---

<details>
<summary><strong>InfluxDB Data Structure</strong></summary>

### Measurements

| Measurement | Key Fields |
|---|---|
| `East Microinverter` | `power_sensor`, `pv_power_sensor`, `pv{1-4}_power/voltage/current_sensor`, `grid_voltage/current/frequency_sensor`, `temperature_sensor`, `today_production_{1-4}_sensor`, `total_production_{1-4}_sensor`, `power_losses_sensor`, `device_state/alarm/fault_sensor` |
| `West Microinverter` | Same structure as East |
| `Smart Meter` | `power_sensor`, `voltage_sensor`, `current_sensor`, `frequency_sensor`, `power_factor_sensor`, `total_import_energy_sensor`, `total_export_energy_sensor` |

### Schema Notes

- Query language: **SQL** (InfluxDB 3 Core — not InfluxQL, not Flux)
- `device_name` and `device_type` are **tag columns** (Dictionary type)
- All numeric fields are `Float64`
- Timestamp column: `time` (nanosecond precision)
- InfluxDB instance: `http://192.168.2.10:8181`, database/bucket: `solar`
- Reporting interval: ~10 seconds

</details>

---

## Dashboard Sections

| Row | Section | What it shows |
|-----|---------|---------------|
| Overview Stats | KPI bar | Total solar power, house load (CT), grid import, self-consumption %, system status |
| Production | Time-series & daily bar | East vs West power over time, daily kWh production this week |
| Module Level | Canvas + bargauge + table | Roof layout heatmap, per-panel power bars, module voltage/current table |
| Inverter Health | State timeline + gauges | Temperature, alarm/fault counters, device state history |
| Grid Status | Stats + time-series | Voltage, frequency, power factor, grid current trends |
| Financial Savings | Stat panels | Daily / this-month / all-time savings in THB |
| Power Distribution | CT panels + reconciliation | Floor 1, Floor 2, and 236 Floor 1 CT load, plus CT total vs derived (grid + solar) comparison |
| Backfeed Log | Table | Timestamped backfeed events from Smart Meter |

## Tech Stack

| Technology | Version / Detail |
|---|---|
| Grafana | 12.4.1 OSS |
| InfluxDB | 3 Core (SQL query mode) |
| Query language | SQL — `DATE_BIN`, window functions, `AT TIME ZONE 'Asia/Bangkok'` |
| Currency / rate | Thai Baht — flat 3.5 THB/kWh |
