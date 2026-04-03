# Phase 9: CT Grid Meter — Grid Import Power, Voltage & Current - Context

**Gathered:** 2026-04-04
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 9 replaces Smart Meter queries with the new `grid` CT meter table for panels that show grid import power, voltage, and current. Panels that require fields not available in the CT table (frequency, power_factor, total_import_energy) remain on the Smart Meter. Backfeed detection panels stay on the Smart Meter because CT measures unidirectional current (always positive).

**In scope:**
- Panel 4 (Grid Import stat) — switch `power_sensor` → `grid`.`power`
- Panel 11 (House ← Grid power flow stat) — switch `power_sensor` → `grid`.`power`
- Panel 16 (Grid Voltage stat) — switch `voltage_sensor` → `grid`.`voltage`
- Panel 19 (Grid Current stat) — switch `current_sensor` → `grid`.`current`
- Panel 20 (Voltage & Frequency time-series) — switch voltage series to `grid`.`voltage`; frequency series stays on Smart Meter
- Panel 21 (Power Factor & Current time-series) — switch current series to `grid`.`current`; power_factor series stays on Smart Meter
- Panel 6 (Self-Consumption stat) — switch its Grid query component from Smart Meter `power_sensor` to `grid`.`power`
- Panel 905 (🔢 Calculated Load reconciliation stat) — switch its Grid query component from Smart Meter `power_sensor` to `grid`.`power`
- Panel 801 (Power Profile time-series, Grid series) — switch Grid series from Smart Meter `power_sensor` to `grid`.`power`

**Out of scope:**
- Panel 17 (Grid Frequency) — stays on Smart Meter `frequency_sensor` (no frequency field in CT table)
- Panel 18 (Power Factor) — stays on Smart Meter `power_factor_sensor` (no power_factor in CT table)
- Panels 33/34/35 (backfeed log) — stay on Smart Meter `power_sensor` (CT is always positive; Smart Meter detects backfeed via power_sensor < 0)
- Panel 20 frequency series — stays on Smart Meter (frequency not in CT table)
- Panel 21 power_factor series — stays on Smart Meter (power_factor not in CT table)
- Any changes to backfeed detection logic

</domain>

<decisions>
## Implementation Decisions

### New CT Grid Measurement

- **D-01:** The new data source is the `grid` table in InfluxDB database `solar`. This is a CT meter placed at the grid connection point, confirmed live since 2026-04-03T17:21 UTC.

- **D-02:** `grid` table field schema (confirmed from live data query):
  | Field | Type | Unit | Notes |
  |-------|------|------|-------|
  | `power` | Float64 | W | Active power, always positive (unidirectional CT) |
  | `current` | Float64 | A | Grid current |
  | `voltage` | Float64 | V | Grid voltage |
  | `energy_today` | Float64 | kWh | Daily energy counter (resets daily) — NOT the same as Smart Meter total_import |
  | `time` | Timestamp | — | Nanosecond precision |

- **D-03:** `grid` table does NOT have: `frequency`, `power_factor`, `total_import_energy`, `total_export_energy`. These remain sourced from Smart Meter for panels that need them.

- **D-04:** No tag columns in the `grid` table (pure field data, same pattern as `235_floor_1`, `235_floor_2`, `air_conditioner`).

- **D-05:** CT at grid point is always positive — it measures current magnitude, not direction. Backfeed detection (`power_sensor < 0`) cannot use this table. Panels 33/34/35 remain on Smart Meter.

- **D-06:** Sample values at 2026-04-03 17:50 UTC: `grid`.power=1330W, `grid`.voltage=237V, `grid`.current=6.55A vs Smart Meter power_sensor=1343W, voltage_sensor=237V, current_sensor=6.60A — close match confirming this is measuring the same grid connection.

### SQL Query Pattern

- **D-07:** Latest-value queries use the same 2-minute recency window pattern established in Phase 7:
  `SELECT power FROM "grid" WHERE time >= now() - INTERVAL '2 minutes' ORDER BY time DESC LIMIT 1`
  (No COALESCE/selector_last needed — use simple ORDER BY DESC LIMIT 1 matching the existing grid pattern)

  **Note:** The `grid` table uses clean field names (`power`, `voltage`, `current`) not `_sensor` suffixed names. Do not add `_sensor` suffix.

- **D-08:** Time-series queries for Panel 20 and Panel 21:
  `SELECT DATE_BIN(INTERVAL '5 minutes', time) AS time, AVG(voltage) AS "Voltage" FROM "grid" WHERE time >= $__timeFrom AND time <= $__timeTo GROUP BY 1 ORDER BY 1`
  (Same DATE_BIN pattern as existing series)

- **D-09:** Power Profile (Panel 801) Grid series:
  `SELECT DATE_BIN(INTERVAL '5 minutes', time) AS time, GREATEST(AVG(power), 0) / 1000.0 AS "Grid" FROM "grid" WHERE time >= $__timeFrom AND time <= $__timeTo GROUP BY 1 ORDER BY 1`
  (Same GREATEST/1000.0 pattern as existing Smart Meter Grid series)

### Mixed-Source Panels (Partial Switch)

- **D-10:** Panel 20 (Voltage & Frequency) has two series:
  - `Voltage` series → switch to `grid`.`voltage`
  - `Frequency` series → stays on Smart Meter `frequency_sensor`
  This is intentional — CT doesn't measure frequency.

- **D-11:** Panel 21 (Power Factor & Current) has two series:
  - `Current` series → switch to `grid`.`current`
  - `Power Factor` series → stays on Smart Meter `power_factor_sensor`
  This is intentional — CT doesn't measure power factor.

### Expression-Based Panels (Panel 6 and Panel 905)

- **D-12:** Panel 6 (Self-Consumption) uses the Expression pattern from Phase 5. Its Grid query target (refId: Grid or similar) should switch from Smart Meter `power_sensor` to `grid`.`power`. The Expression formula `$Solar / $Total` where `$Total = $Solar + $Grid` remains unchanged.

- **D-13:** Panel 905 (🔢 Calculated Load) uses Expression `$East + $West + $Grid`. Its Grid query target switches from Smart Meter `power_sensor` to `grid`.`power`. Expression formula unchanged.

### Agent's Discretion
- Whether to use `COALESCE(... 0)` or `GREATEST(... 0)` for the power field on Grid Import (D-07) — follow whichever pattern the existing grid power queries use
- Exact refId names for the new CT query targets (use descriptive names like "GridCT" or "Grid")
- Whether Panel 20's two series need any display label updates

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project Specs
- `.planning/PROJECT.md` — InfluxDB schema, datasource name `influxdb-solar`, system context, Key Decisions table
- `.planning/REQUIREMENTS.md` — GRID-01, GRID-02 requirements for grid panels

### Existing Dashboard
- `solar-pv-monitor.json` — The dashboard to be modified. All panel edits go here.

### Prior Phase Context
- `.planning/phases/07-fix-stale-data-when-inverter-goes-offline-at-sunset/07-01-PLAN.md` — 2-minute recency window pattern for latest-value queries; apply same pattern to all CT grid queries
- `.planning/phases/05-fix-all-panel-in-dashboard-that-still-use-transformation-to-calcuate-to-use-expression-instead-for-example-look-at-self-consumption-or-house-load-panel-that-use-expression/05-CONTEXT.md` — Expression target JSON structure for panels 6 and 905
- `.planning/phases/08-update-house-load-measurement-to-use-ct-meters-on-both-consumer-units-add-per-floor-power-distribution-panels-and-data-reconciliation-panel-comparing-calculated-grid-solar-vs-actual-ct-measurement/08-CONTEXT.md` — CT meter field naming convention (clean names: `power`, `voltage`, `current` — no `_sensor` suffix)

### InfluxDB Live Schema (verified 2026-04-04)
- `grid` table confirmed present in `solar` database
- Fields: `power` (W), `current` (A), `voltage` (V), `energy_today` (kWh)
- No tag columns
- Data started: 2026-04-03T17:21 UTC
- Sample values at 2026-04-03 17:50: power=1330W, voltage=237V, current=6.55A

</canonical_refs>

<code_context>
## Existing Code Insights

### Panels Affected and Their Current SQL

| Panel | Current Smart Meter SQL | Switch field | New field |
|-------|------------------------|-------------|-----------|
| 4 (Grid Import) | `power_sensor FROM "Smart Meter"` | power_sensor | `power FROM "grid"` |
| 11 (House ← Grid) | `power_sensor FROM "Smart Meter"` | power_sensor | `power FROM "grid"` |
| 16 (Grid Voltage) | `voltage_sensor FROM "Smart Meter"` | voltage_sensor | `voltage FROM "grid"` |
| 19 (Grid Current) | `current_sensor FROM "Smart Meter"` | current_sensor | `current FROM "grid"` |
| 20 (V&F time-series) | `voltage_sensor` + `frequency_sensor` | voltage_sensor only | `voltage FROM "grid"` |
| 21 (PF&C time-series) | `power_factor_sensor` + `current_sensor` | current_sensor only | `current FROM "grid"` |
| 6 (Self-Consumption) | Grid target: `power_sensor FROM "Smart Meter"` | power_sensor | `power FROM "grid"` |
| 905 (Calc Load) | Grid target: `power_sensor FROM "Smart Meter"` | power_sensor | `power FROM "grid"` |
| 801 (Power Profile) | Grid series: `power_sensor FROM "Smart Meter"` | power_sensor | `power FROM "grid"` |

### Panels That STAY on Smart Meter

| Panel | Smart Meter Field | Reason |
|-------|-------------------|--------|
| 17 (Grid Frequency) | `frequency_sensor` | CT has no frequency field |
| 18 (Power Factor) | `power_factor_sensor` | CT has no power_factor field |
| 20 (Frequency series) | `frequency_sensor` | CT has no frequency field |
| 21 (Power Factor series) | `power_factor_sensor` | CT has no power_factor field |
| 33 (Backfeed Count) | `power_sensor < 0` | CT is always positive |
| 34 (Max Backfeed) | `power_sensor < 0` | CT is always positive |
| 35 (Backfeed Log) | `power_sensor`, `voltage_sensor`, `current_sensor` | Backfeed detection requires signed power |

### Established Patterns
- **CT field names**: `power`, `voltage`, `current` (no `_sensor` suffix — same as `235_floor_1`/`235_floor_2`)
- **Latest-value**: `WHERE time >= now() - INTERVAL '2 minutes' ORDER BY time DESC LIMIT 1`
- **Time-series**: `DATE_BIN(INTERVAL '5 minutes', time) AS time, AVG(field) ... GROUP BY 1 ORDER BY 1`
- **Power Profile kW scale**: divide by 1000.0, use `GREATEST(..., 0)` to clip negatives for grid import

### Integration Points
- All changes are in-place SQL replacements in existing panel target objects
- No new panels, no row additions, no gridPos changes
- Total affected panels: 9 (4 single-source panels, 2 partial-source panels, 3 expression-component panels)

</code_context>

<specifics>
## Specific Ideas

- User confirmed the `grid` table is the new CT meter data source for Phase 9
- CT at grid connection point is always positive (unidirectional) — confirmed by live data showing no negative values
- `energy_today` field in `grid` table is a daily-reset counter (unlike `235_floor_1/2` which is cumulative since install) — may be useful for "Today's Grid Import" stat in a future phase, but not in scope for Phase 9
- `powct-20260403T172015` table also exists with same schema as `air_conditioner` — appears to be a test/staging artifact from the same day; treat it as irrelevant to this phase

</specifics>

<deferred>
## Deferred Ideas

- **Today's Grid Import energy stat** — `grid`.`energy_today` could power a new stat panel showing daily imported kWh. Not in Phase 9 scope.
- **Air conditioner monitoring** — `air_conditioner` table exists with power/voltage/current. Could add an AC power stat panel to the Power Distribution row. Separate phase.

</deferred>

---

*Phase: 09-update-panels-to-use-new-ct-meter-data-for-grid-energy-import-current-voltage-real-power-instead-of-smart-meter*
*Context gathered: 2026-04-04*
