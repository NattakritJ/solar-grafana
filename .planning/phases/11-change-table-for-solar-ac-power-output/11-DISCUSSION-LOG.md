# Phase 11: Change table for solar AC power output - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-09
**Phase:** 11-change-table-for-solar-ac-power-output
**Areas discussed:** Panel migration scope, Energy / today counter, Panels with no CT equivalent, Recency window tightening, Dashboard refresh interval

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

---

## Recency Window for NRT Snapshot Panels

| Option | Description | Selected |
|--------|-------------|----------|
| Keep 1-minute window | All CT snapshot panels stay at `INTERVAL '1 minutes'` | |
| Tighten to 10 seconds | All CT snapshot panels use `INTERVAL '10 seconds'` | ✓ |
| Tighten to 5 seconds | Aggressive — may only capture 2-3 readings | |

**User's choice:** Tighten all NRT snapshot panels to `INTERVAL '10 seconds'`. This applies to all 15 snapshot panels querying CT tables (10 pre-existing + 3 newly migrated + 2 hybrid panels). At ~2s CT log intervals, a 10-second window captures ~5 readings for AVG — sufficient for smoothing while being much more responsive than the old 1-minute window.

**Notes:** User added this as a second scope item after the initial CT table migration discussion. The rationale is that since CT tables log at ~2s, a 1-minute window averages over ~30 readings which is unnecessarily stale for real-time monitoring. Time-series panels (801, 20, 21, 12) and backfeed panels (33, 34, 35) are unaffected — they use `$__timeFrom`/`$__timeTo` range queries, not `now() - INTERVAL` recency windows.

---

## Dashboard Refresh Interval

| Option | Description | Selected |
|--------|-------------|----------|
| Keep 10s refresh | Dashboard refreshes every 10 seconds | |
| Change to 5s refresh | Dashboard refreshes every 5 seconds | ✓ |
| Change to 2s refresh | Match CT log rate directly | |

**User's choice:** Change dashboard refresh from `10s` → `5s`. The faster CT log rate (~2s) means the dashboard can update more frequently. A 5-second refresh balances responsiveness with browser/server load.

**Notes:** JSON change is simply `"refresh": "10s"` → `"refresh": "5s"` at the dashboard root level.

---

## Inverter-Only Panel Window Treatment

| Option | Description | Selected |
|--------|-------------|----------|
| Tighten inverter panels too | Change 1-min → 10s on inverter panels | |
| Leave inverter panels unchanged | Keep existing windows on all inverter-querying panels | ✓ |
| Mixed approach | Tighten some, leave others | |

**User's choice:** Leave all inverter-only snapshot panels (23, 24, 25, 26-31 with 1-min; 8, 22, 45 with 5-min) unchanged. The inverter logs at ~60s intervals, so a 10-second window would frequently miss data and show nulls/zeros.
