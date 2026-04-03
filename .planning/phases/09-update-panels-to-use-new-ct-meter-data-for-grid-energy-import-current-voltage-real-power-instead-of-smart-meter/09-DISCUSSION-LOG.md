# Phase 9: CT Grid Meter — Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-04
**Phase:** 09-update-panels-to-use-new-ct-meter-data-for-grid-energy-import-current-voltage-real-power-instead-of-smart-meter
**Areas discussed:** New CT meter identity, Which panels get updated, Grid Import power sign/backfeed, Self-Consumption panel, Calculated Load panel

---

## Pre-Discussion: Live InfluxDB Data Check

User requested live data check before discussion. Queried InfluxDB at 192.168.2.10:8181.

**New tables discovered (not in prior phases):**
- `grid` — CT meter at grid connection point, started 2026-04-03T17:21 UTC
- `air_conditioner` — AC power monitoring
- `powct-20260403T172015` — temporary/test artifact

**`grid` table schema confirmed:**
- Fields: `power` (W), `current` (A), `voltage` (V), `energy_today` (kWh)
- No tag columns, no `_sensor` suffix (same pattern as `235_floor_1/2`)
- Sample values: power=1330W, voltage=237V, current=6.55A

**Smart Meter for comparison:** power_sensor=1343W, voltage_sensor=237V, current_sensor=6.60A — close match confirming same grid connection point.

---

## New CT Meter Identity

| Option | Description | Selected |
|--------|-------------|----------|
| `grid` table (new CT at grid point) | Started 2026-04-03T17:21 UTC. Fields: power, current, voltage, energy_today. Confirmed match to Smart Meter readings. | ✓ |
| Something else | Different unidentified data source | |

**User's choice:** `grid` table (new CT at grid point)
**Notes:** Confirmed by live data comparison showing close values to Smart Meter.

---

## Which Panels Get Updated

| Option | Description | Selected |
|--------|-------------|----------|
| Power panels only (3 panels) | Panel 4, 11, 801 | |
| Power + Voltage + Current panels (7 panels) | Above + Panel 16, 19, 20, 21 | |
| All Smart Meter panels (replace everything) | Including frequency, power_factor, backfeed | |
| Partial switch for mixed panels | Switch power/voltage/current to CT; keep frequency/power_factor/total_import on Smart Meter | ✓ |

**User's choice:** Switch to CT for all fields the CT table provides (power, voltage, current); keep frequency, power_factor, total_import_energy on Smart Meter.
**Notes:** User explicitly stated "keep using data that doesn't provide by CT meter (frequency, power factor, total_import_energy) from current smart meter."

---

## Grid Import Power Sign / Backfeed

| Option | Description | Selected |
|--------|-------------|----------|
| CT always positive — no sign issue | Backfeed panels (33/34/35) stay on Smart Meter | ✓ |
| CT may go negative — keep backfeed on Smart Meter | Check for negative CT values | |

**User's choice:** CT is always positive — no sign issue
**Notes:** Backfeed log panels (33/34/35) remain on Smart Meter because CT measures unidirectional current.

---

## Self-Consumption Panel (Panel 6)

| Option | Description | Selected |
|--------|-------------|----------|
| Switch to CT grid table | Grid query component switches to `grid`.`power`; formula unchanged | ✓ |
| Keep on Smart Meter | Leave panel 6 unchanged | |

**User's choice:** Switch to CT grid table (consistent with panel 4)
**Notes:** Expression formula Solar / (Solar + Grid) stays the same; only the Grid query target changes.

---

## Calculated Load Panel (Panel 905)

| Option | Description | Selected |
|--------|-------------|----------|
| Switch Panel 905 to CT grid table | Grid query component switches to `grid`.`power`; formula unchanged | ✓ |
| Keep Panel 905 on Smart Meter | Preserves cross-source reconciliation visibility | |

**User's choice:** Switch Panel 905 to CT grid table
**Notes:** Expression formula East + West + Grid stays the same; only the Grid query target changes.

---

## Agent's Discretion

- Exact refId names for new CT query targets
- Whether to use COALESCE or GREATEST for power field clipping
- Panel 20 display label updates if needed

## Deferred Ideas

- **Today's Grid Import energy stat** — `grid`.`energy_today` could be a new stat panel. Not Phase 9 scope.
- **Air conditioner monitoring** — `air_conditioner` table exists, could add AC panel. Separate phase.
