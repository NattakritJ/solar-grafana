# Phase 11: Change table for solar AC power output - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-09
**Phase:** 11-change-table-for-solar-ac-power-output
**Areas discussed:** Panel migration scope, Energy / today counter, Panels with no CT equivalent

---

## Clarifying Question: Phase Intent

| Option | Description | Selected |
|--------|-------------|----------|
| New InfluxDB table | New measurement for AC output, replace queries | ✓ |
| Rename existing table | Measurement name changed | |
| Change Grafana table panel | Column changes to Module Detail | |
| Use pv_power_sensor field | Switch DC/AC field | |

**User's choice:** New CT sensors (`east_microinverter_power`, `west_microinverter_power`) were added to measure AC output directly from inverter output cables. Replaces inverter measurement (`power_sensor`) for AC power queries because the inverter logs slowly (~60s, stale values).

**Notes:** User also provided InfluxDB connection credentials and noted the new CT tables use the same schema as the existing load CT tables (`power`, `voltage`, `current`, `energy`, `frequency`, `power_factor`).

---

## Panel Migration Scope

| Option | Description | Selected |
|--------|-------------|----------|
| All 9 AC power panels | All panels using East/West Microinverter power_sensor | ✓ |
| Real-time stat panels only | Only stat/bargauge panels, leave time-series | |
| Primary display panels only | Just panel 1 and 9 | |

**User's choice:** All 9 AC power panels (later refined to 8 after discovering panel 46 uses `pv_power_sensor` DC, not `power_sensor` AC).

**Notes:** Agent discovered panel 46 uses `pv_power_sensor` (DC aggregated) not `power_sensor` (AC output) — no CT equivalent exists, so panel 46 stays on inverter. Effective count: 8 panels.

---

## Energy / Today Counter

| Option | Description | Selected |
|--------|-------------|----------|
| Migrate to CT energy field | Use MAX(energy) from CT tables for today's energy | |
| Keep on inverter today_production | Keep today_production_sensor from East/West Microinverter | ✓ |
| Verify behavior first | New table just started, need to confirm daily reset | |

**User's choice:** Keep Today's Free Energy (panel 2) and savings panels (38/39/40) on `today_production_sensor` from inverter measurements.

**Notes:** At context-gathering time, the new CT tables had only ~10 minutes of data (started 2026-04-08T23:26:45). The `energy` field behavior (daily reset vs. total cumulative) could not be verified. User chose to keep proven inverter-based energy counters in this phase.

---

## Panels with No CT Equivalent

| Option | Description | Selected |
|--------|-------------|----------|
| Keep DC/per-panel on inverter | Panel 46, 22, 23, 45 all stay on inverter | ✓ |
| Switch panel 46 to CT power | Rename and switch heatmap to AC data | |
| Remove panel 46 | Remove heatmap as redundant | |

**User's choice:** Keep all DC/per-panel panels on inverter tables. The per-panel `pv{1-4}` DC channels have no CT equivalent, so Module Detail (23), Panel DC Power (22), Canvas (45), and Daily PV DC Heatmap (46) all stay on inverter measurements unchanged.

---

## Agent's Discretion

- Exact sequencing of the 8 panel updates
- Whether to use a single scan-and-replace approach or panel-by-panel
- Verification strategy (test InfluxDB query before/after each panel)

## Deferred Ideas

- CT energy field migration for today's energy — needs multi-day data to verify daily reset behavior
- air_conditioner table monitoring (from Phase 10 deferred)
- CT energy for lifetime total production (CT tables have no lifetime counter)
