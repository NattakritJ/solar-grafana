# Phase 11: Change table for solar AC power output - Context

**Gathered:** 2026-04-09
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 11 migrates solar AC power output queries from the slow-polling `"East Microinverter"` / `"West Microinverter"` measurements to two new dedicated CT sensor tables: `east_microinverter_power` and `west_microinverter_power`.

**Root cause:** The inverter measurements log at ~60-second intervals and the logger polls aggressively, causing the device to report stale/repeated values. The new CT sensors clamp directly on the inverter AC output cables and log at ~2s intervals (same as other CT load tables).

**In scope:**
- Migrate all 9 panels that query `power_sensor` from East/West Microinverter for solar AC output to the new CT tables
- New CT field for power: `power` (no `_sensor` suffix — same schema as `235_floor_1`, `235_floor_2`, `236_floor_1`)
- All changes in `solar-pv-monitor.json` only

**Out of scope:**
- Today's Free Energy (panel 2) and savings panels (38/39/40) — stay on `today_production_sensor` from East/West Microinverter
- Module Detail (panel 23) — stays on inverter `pv{1-4}_power/voltage/current_sensor` (DC input channels, no CT equivalent)
- Panel DC Power bargauge (panel 22) — stays on inverter DC channel fields
- Canvas roof layout (panel 45) — stays on inverter `pv{1-4}_power_sensor` DC channels
- Daily PV DC Power Heatmap (panel 46) — stays on inverter `pv_power_sensor` (DC aggregated, no CT equivalent)
- Inverter health panels (temperature, state/alarm/fault) — no CT equivalent, stay on inverter

</domain>

<decisions>
## Implementation Decisions

### New CT Table Schema

- **D-01:** New tables:
  - `east_microinverter_power` — East inverter AC output CT sensor
  - `west_microinverter_power` — West inverter AC output CT sensor
  
  Schema (verified from live InfluxDB):
  | Field | Type | Notes |
  |-------|------|-------|
  | `power` | Float64 | AC output power (W) — replaces `power_sensor` |
  | `voltage` | Float64 | AC voltage (V) |
  | `current` | Float64 | AC current (A) |
  | `energy` | Float64 | Daily energy counter (Wh) |
  | `frequency` | Float64 | AC frequency (Hz) |
  | `power_factor` | Float64 | Power factor |
  | `time` | Timestamp(ns) | — |

  No tag columns (no `device_name`, `device_type`) — unlike the inverter measurements.

- **D-02:** Field name mapping: `power_sensor` → `power` (no suffix). The `COALESCE(AVG(power_sensor), 0)` pattern becomes `COALESCE(AVG(power), 0)`.

- **D-03:** Time window stays at `now() - INTERVAL '1 minutes'` — same as the current recency window applied to these panels in the existing dashboard (note: some panels use 1 minute, not 2 minutes — keep existing window per panel).

### Panel Migration List (9 panels)

- **D-04:** These 9 panels migrate their East/West solar power queries from inverter tables to CT tables:

| Panel | Title | Change |
|-------|-------|--------|
| 1 | Solar AC Power | `"East Microinverter"` → `east_microinverter_power`, `power_sensor` → `power` |
| 6 | Self-Consumption | East/West hidden targets → CT tables |
| 7 | Peak AC Power Today | `"East Microinverter"` → CT, `MAX(power_sensor)` → `MAX(power)` |
| 9 | ☀️ Solar AC → House | East/West targets → CT tables |
| 12 | Solar AC Production | Time-series East/West → CT tables |
| 13 | Inverter AC Power | bargauge East/West → CT tables |
| 46 | Daily PV DC Power Heatmap | **EXCEPTION — see D-05** |
| 801 | Power Profile | Production series → CT tables |
| 905 | 🔢 Calculated Load | East/West hidden targets → CT tables |

- **D-05:** Panel 46 ("Daily PV DC Power Heatmap") is in the initial grep output but uses `pv_power_sensor` (DC aggregated total), **not** `power_sensor` (AC output). It has **no CT equivalent** — keep on `"East Microinverter"` / `"West Microinverter"` unchanged. The effective migration list is **8 panels** (46 excluded).

### Energy / Today Counter

- **D-06:** Today's Free Energy (panel 2) and financial savings panels (38/39/40) stay on `today_production_sensor` / `today_production_{1-4}_sensor` from the East/West Microinverter measurements. These are accurate inverter-reported daily cumulative counters. The CT `energy` field is brand new (started today) and its daily-reset behavior is unverified — do not migrate these panels in Phase 11.

### DC / Per-Panel Panels (No Migration)

- **D-07:** The following panels have no CT equivalent and must stay on inverter measurements unchanged:
  - Panel 22: Panel DC Power (uses `pv{1-4}_power_sensor` per-channel DC)
  - Panel 23: Module Detail (uses `pv{1-4}_power/voltage/current_sensor` DC channels)
  - Panel 45: Canvas Roof Layout (uses `pv{1-4}_power_sensor` DC for heatmap colors)
  - Panel 46: Daily PV DC Power Heatmap (uses `pv_power_sensor` DC aggregated)
  - Panels 24/25: East/West Temperature (uses `temperature_sensor`)
  - Panels 26-31: State/Alarm/Fault stats (uses `device_state/alarm/fault_sensor`)

### Agent's Discretion

- Whether to process the 8 panels one-by-one in separate plan steps or batch into fewer broader replacements
- The exact `refId` alias convention in the new SQL strings (keep existing aliases like `"East"`, `"West"` unchanged for Expression compatibility)
- Whether to verify with an InfluxDB test query before modifying each panel

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project Specs
- `.planning/PROJECT.md` — InfluxDB schema, datasource name `influxdb-solar`, system context, Key Decisions table
- `.planning/REQUIREMENTS.md` — requirements context

### Existing Dashboard
- `solar-pv-monitor.json` — The dashboard to be modified. All changes go here.

### Prior Phase Context
- `.planning/phases/10-after-using-dashboard-for-a-while-i-found-a-major-problem-each-device-have-different-log-time-and-almost-all-query-use-desc-limit-1-so-data-coming-to-dashboard-are-not-in-the-same-timeframe-how-can-we-tackle-this-problem/10-CONTEXT.md` — Phase 10 established AVG over recency window for all snapshot queries; Phase 11 continues this pattern
- `.planning/phases/09-update-panels-to-use-new-ct-meter-data-for-grid-energy-import-current-voltage-real-power-instead-of-smart-meter/09-CONTEXT.md` — CT table migration precedent (grid table); same `power`/`voltage`/`current` field names (no `_sensor` suffix)
- `.planning/phases/08-update-house-load-measurement-to-use-ct-meters-on-both-consumer-units-add-per-floor-power-distribution-panels-and-data-reconciliation-panel-comparing-calculated-grid-solar-vs-actual-ct-measurement/08-CONTEXT.md` — CT field naming convention established (`power`, `voltage`, `current` — no `_sensor` suffix)

### InfluxDB Live Schema (verified 2026-04-09)

New CT table characteristics confirmed from live InfluxDB:

| Table | Log Interval | Started | Fields |
|-------|-------------|---------|--------|
| `east_microinverter_power` | ~2s | 2026-04-08T23:26:45 | power, voltage, current, energy, frequency, power_factor |
| `west_microinverter_power` | ~2s | 2026-04-08T23:26:45 | power, voltage, current, energy, frequency, power_factor |

Connection: `http://192.168.2.10:8181`, database: `solar`, datasource: `influxdb-solar`

</canonical_refs>

<code_context>
## Existing Code Insights

### Current AC Power Query Pattern (to be replaced)
```sql
-- Old: inverter measurement, slow ~60s interval
SELECT COALESCE(AVG(power_sensor), 0) AS solar_kw
FROM "East Microinverter"
WHERE time >= now() - INTERVAL '1 minutes'
```

### New AC Power Query Pattern
```sql
-- New: CT measurement, fast ~2s interval
SELECT COALESCE(AVG(power), 0) AS solar_kw
FROM east_microinverter_power
WHERE time >= now() - INTERVAL '1 minutes'
```

Note: CT table names are **unquoted** (no spaces) — `east_microinverter_power`, `west_microinverter_power`. Inverter tables used double-quoted names with spaces: `"East Microinverter"`.

### Panels Migrating (8 panels — panel 46 excluded)

| Panel | Title | Targets to change |
|-------|-------|------------------|
| 1 | Solar AC Power | 2 targets (East + West hidden) |
| 6 | Self-Consumption | 2 targets (East, West in hidden set) |
| 7 | Peak AC Power Today | 2 targets (East + West) |
| 9 | ☀️ Solar AC → House | 2 targets (East + West hidden) |
| 12 | Solar AC Production | 2 targets (East, West time-series) |
| 13 | Inverter AC Power | 2 targets (East, West bargauge) |
| 801 | Power Profile | East+West production targets |
| 905 | 🔢 Calculated Load | East+West hidden targets |

### Panels Explicitly NOT Changing

| Panel | Title | Reason |
|-------|-------|--------|
| 2 | Today's Free Energy | Uses `today_production_sensor` — daily counter, stays on inverter |
| 22 | Panel DC Power | Uses `pv{1-4}_power_sensor` DC — no CT equivalent |
| 23 | Module Detail | Uses DC per-panel channels — no CT equivalent |
| 38/39/40 | Financial Savings | Use `today_production_N_sensor` — stays on inverter |
| 45 | Roof Layout Canvas | Uses DC `pv{1-4}_power_sensor` per panel |
| 46 | Daily PV DC Heatmap | Uses `pv_power_sensor` (DC aggregated) — no CT equivalent |
| 24/25 | Temperature | No CT equivalent |
| 26-31 | State/Alarm/Fault | No CT equivalent |

### Established Patterns
- CT table precedent set by Phase 9 (`grid` table) and Phase 8 (`235_floor_1`, `235_floor_2`, `236_floor_1`)
- All CT tables use `power`, `voltage`, `current` (no `_sensor` suffix)
- `COALESCE(AVG(field), 0)` wrapper stays — handles no-data during the 2s log gap
- Expression (math) targets that sum East+West (e.g., in panels 1, 6, 9, 905) are unaffected — they reference refIds whose underlying SQL changes

### Integration Points
- All changes are in-place SQL replacements in `rawSql` strings within `solar-pv-monitor.json`
- Expression math targets are unaffected (they reference `$East + $West` etc.)
- No new panels, no gridPos changes, no row additions

</code_context>

<specifics>
## Specific Ideas

- Live data comparison at time of context gathering (nighttime, post-sunset):
  - `"East Microinverter".power_sensor` via `selector_last`: 43.6W (frozen — stale inverter reading)
  - `east_microinverter_power.power` via latest reading: 141-143W (live CT, updating every 2s)
  - This confirms the problem: inverter logs ~60s intervals with stale/frozen values; CT is live
- New tables just started (2026-04-08T23:26:45) — only ~10 minutes of historical data at context-gathering time
- The `energy` field in new CT tables is NOT verified as a daily-reset counter (only 10 min of data) — do not migrate energy panels in this phase

</specifics>

<deferred>
## Deferred Ideas

- **Migrate Today's Free Energy and savings to CT energy field** — The CT `energy` field may be a daily cumulative counter like `today_production_sensor`, but it only has 10 minutes of data as of Phase 11. Revisit in a future phase once daily reset behavior is confirmed over multiple days.
- **air_conditioner monitoring** — `air_conditioner` table noted in Phase 10 deferred ideas. Still out of scope.
- **CT energy for lifetime total** — The CT tables don't have a total lifetime counter like `total_production_sensor`. No migration path for lifetime energy panels.

</deferred>

---

*Phase: 11-change-table-for-solar-ac-power-output*
*Context gathered: 2026-04-09*
