# Phase 8: CT Meters, Per-Floor Distribution & Reconciliation - Context

**Gathered:** 2026-04-02
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 8 integrates newly-installed CT (current transformer) meters into the dashboard. It replaces the derived house load calculation (solar + grid) with direct CT measurement, adds a new Power Distribution section showing per-floor power, and adds reconciliation stat panels to compare the CT-measured load against the derived calculation.

**In scope:**
- Replace House Load panels 5 and 10 to use CT direct measurement
- Add new "Power Distribution" row with Floor 1 and Floor 2 current power stat panels
- Add CT Load and Calculated Load reconciliation stat panels in the Power Distribution row
- Update Power Profile time-series (panel 801) to include Floor 1 and Floor 2 series

**Out of scope:**
- Changing the Self-Consumption panel formula (stays as Solar ÷ (Solar + Grid))
- Per-floor daily/cumulative energy stats from CT (energy counter is not day-reset; smart meter remains the source for daily energy)
- CT alarm threshold monitoring
- Calibration workflows

</domain>

<decisions>
## Implementation Decisions

### CT Data Source

- **D-01:** Two CT measurements in InfluxDB (database: `solar`):
  - `235_floor_1` — Floor 1 consumer unit (PZEM-016, 100A CT)
  - `235_floor_2` — Floor 2 consumer unit (PZEM-016, 100A CT)

- **D-02:** CT field schema (identical for both measurements):
  | Field | Type | Unit | Notes |
  |-------|------|------|-------|
  | `power` | Float64 | W | Active power, 0.1W resolution, 0.5% accuracy |
  | `energy` | Float64 | Wh | Cumulative energy counter since install (NOT daily) |
  | `current` | Float64 | A | 0.001A resolution |
  | `voltage` | Float64 | V | 0.1V resolution |
  | `frequency` | Float64 | Hz | 0.1Hz resolution |
  | `power_factor` | Float64 | 0-1 | 0.01 resolution |
  | `time` | Timestamp | — | Nanosecond precision |

- **D-03:** CT meters report every ~10 seconds (same interval as inverters). No tag columns — pure field data.

- **D-04:** The `energy` field is a **cumulative counter** since the meter was physically installed/reset — NOT a daily reset counter. Do NOT use `energy` for "today's energy" calculations. Daily energy for the house remains sourced from the smart meter.

- **D-05:** Apply the established Phase 7 pattern: `WHERE time >= now() - INTERVAL '2 minutes'` on all CT latest-value queries so panels show no-data if CT meters go offline.

### House Load Panels — Replace with CT Direct Measurement

- **D-06:** **Panel 5 (House Load stat, overview row)** and **Panel 10 (🏠 House Load, power flow row)** switch from the derived formula to CT direct measurement:
  - OLD: `$East + $West + $Grid` via Expression targets
  - NEW: `floor1.power + floor2.power` via two InfluxDB queries + Expression `$Floor1 + $Floor2`
  - SQL pattern (per floor): `SELECT power FROM "235_floor_1" WHERE time >= now() - INTERVAL '2 minutes' ORDER BY time DESC LIMIT 1`

- **D-07:** Use the same Expression target pattern established in Phase 5: two raw InfluxDB queries (refId: Floor1, Floor2) with `hide: true`, plus an Expression target `$Floor1 + $Floor2`. Remove the old East/West/Grid query targets.

- **D-08:** Panel 6 (Self-Consumption) stays **unchanged**. Formula `$Solar / $Total` where `$Total = $Solar + $Grid` is mathematically equivalent to CT for a zero-export system, and relies on the proven smart meter rather than newly-installed CT hardware.

### Per-Floor Power Distribution Panels

- **D-09:** Add a new **"Power Distribution"** row section between the existing Overview section (y≈0–12) and the Power Profile section (row 800, y=12). This requires shifting row 800 and all downstream rows down by the height of the new section.

- **D-10:** New panels in the Power Distribution row:
  - **Floor 1 Power** — stat panel, unit: W (`watt`), query: `SELECT power FROM "235_floor_1" WHERE time >= now() - INTERVAL '2 minutes' ORDER BY time DESC LIMIT 1`
  - **Floor 2 Power** — stat panel, unit: W (`watt`), query: `SELECT power FROM "235_floor_2" WHERE time >= now() - INTERVAL '2 minutes' ORDER BY time DESC LIMIT 1`

- **D-11:** Color scheme for floor power stats: orange (same convention as House Load panels — consumption/load = orange per existing D-14 color convention from Phase 1).

- **D-12:** No "today energy" stat panels from CT data. Energy stats remain solely from the smart meter's existing panels.

### Reconciliation Panels

- **D-13:** Add **two reconciliation stat panels** in the Power Distribution row alongside the floor power panels:
  - **CT Load** — `floor1.power + floor2.power` (W) — uses same CT queries as House Load panels (D-06). Uses Expression `$Floor1 + $Floor2`.
  - **Calculated Load** — `east.power + west.power + grid.power` (W) — uses the same Expression formula as the old House Load panels. Queries: East Microinverter `power_sensor`, West Microinverter `power_sensor`, Smart Meter `power_sensor`, all with 2-minute recency window.

- **D-14:** Purpose: side-by-side visual comparison of the two methods so the homeowner can see how closely the CT measurement tracks the derived calculation. No delta/deviation calculation needed — the user can read the difference at a glance.

- **D-15:** Unit for reconciliation panels: W (`watt`). Color scheme: neutral (use default stat color or match House Load orange — agent's discretion).

### Power Profile Time-Series Update

- **D-16:** Add two new series to **Panel 801 (Power Profile)** time-series:
  - `235 Floor 1` — `SELECT DATE_BIN(INTERVAL '5 minutes', time) AS time, AVG(power) / 1000.0 AS "235 Floor 1" FROM "235_floor_1" WHERE time >= $__timeFrom AND time <= $__timeTo GROUP BY 1 ORDER BY 1`
  - `235 Floor 2` — same pattern for `235_floor_2`
  - Unit: kW (divide by 1000 to match existing Production/Grid/Load series)
  - Note: field name is `power` (not `power_sensor`) for CT measurements

- **D-17:** CT data in Power Profile started 2026-04-02 (~11:34 UTC). Historical time ranges before this date will show no data for Floor 1/Floor 2 series — this is expected behavior.

### Agent's Discretion
- Exact gridPos values (x, y, h, w) for new panels — plan based on dashboard layout at time of execution
- Color of CT Load / Calculated Load reconciliation panels
- Whether to add a label or description text to the Power Distribution row panel
- Whether the Power Distribution row gets a specific row ID number (suggested: 900 to follow existing pattern of 100, 200, 300... 800)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project Specs
- `.planning/PROJECT.md` — InfluxDB schema (measurement names, field names), datasource name `influxdb-solar`, system hardware context
- `.planning/REQUIREMENTS.md` — Requirements context for affected panels

### CT Meter Hardware
- `ct_datasheet.txt` — PZEM-014/016 datasheet confirming field units: power (W, 0.1W res), energy (Wh cumulative), current (A), voltage (V), frequency (Hz), power_factor. Energy range 0–9999.99kWh, can be reset via software.

### Existing Dashboard
- `solar-pv-monitor.json` — The dashboard to be modified. Single file for all panel edits.

### Prior Phase Context
- `.planning/phases/05-fix-all-panel-in-dashboard-that-still-use-transformation-to-calcuate-to-use-expression-instead-for-example-look-at-self-consumption-or-house-load-panel-that-use-expression/05-CONTEXT.md` — D-04 through D-08: Expression target JSON structure, merge removal pattern, how House Load and Self-Consumption panels currently work
- `.planning/phases/07-fix-stale-data-when-inverter-goes-offline-at-sunset/07-01-PLAN.md` — Documents the 2-minute recency window pattern applied to all latest-value queries; CT queries must follow the same pattern

### InfluxDB Live Schema (verified 2026-04-02)
- `235_floor_1` and `235_floor_2` confirmed present in `solar` database with fields: power, energy, current, voltage, frequency, power_factor, time
- Current power values at time of context gathering: floor_1 ~1178W, floor_2 ~590W
- Data started: 2026-04-02T11:34 UTC (brand new installation)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- **Panel 5 (House Load) current target structure** — has East/West/Grid InfluxDB queries + Expression `$East + $West + $Grid`. Replace East/West/Grid queries with Floor1/Floor2 CT queries; update Expression to `$Floor1 + $Floor2`. Same JSON structure.
- **Panel 10 (🏠 House Load power flow)** — same structure as Panel 5; apply identical change.
- **Panel 6 Self-Consumption target structure** — already has Solar/Total/Self Expression targets; LEAVE UNCHANGED as confirmed in discussion.
- **Power Profile (Panel 801)** — existing Production, Grid, Load series use `DATE_BIN(INTERVAL '5 minutes', time) ... GROUP BY 1 ORDER BY 1` pattern. CT queries follow the same pattern but with `power` field instead of `power_sensor`, and no UNION ALL needed (single measurement per series).
- **Phase 5 Expression pattern** — raw query targets use `hide: true`, Expression target datasource: `{"type": "__expr__", "uid": "__expr__"}`, `type: "math"`, `expression: "$A + $B"`

### Established Patterns
- **Latest-value query**: `SELECT field FROM "measurement" WHERE time >= now() - INTERVAL '2 minutes' ORDER BY time DESC LIMIT 1`
- **Time-series query**: `SELECT DATE_BIN(INTERVAL '5 minutes', time) AS time, AVG(field) / 1000.0 AS "SeriesName" FROM "measurement" WHERE time >= $__timeFrom AND time <= $__timeTo GROUP BY 1 ORDER BY 1`
- **Color convention**: load/consumption = orange; solar production = green; grid = blue
- **Row ID convention**: 100, 200, 300, 400, 500, 600, 700, 800 — new Power Distribution row → suggested ID 900
- **Units**: `watt` for W stats, `kwatt` for kW time-series (existing Power Profile uses `kW` suffix in series name with divide-by-1000)

### Integration Points
- **New "Power Distribution" row** must be inserted between Overview section (ends ~y=12) and Power Profile row (row 800, currently y=12). Inserting rows requires shifting y-positions of row 800 and all panels below it.
- **Panels 5 and 10** — in-place target replacement. No gridPos changes.
- **Panel 801 (Power Profile)** — add two new target entries to `targets[]` array. No gridPos changes.
- Dashboard currently has 42 non-row panels (as of Phase 7 completion with 43 total non-row panels confirmed in 07-01-PLAN verification).

</code_context>

<specifics>
## Specific Ideas

- User confirmed CT meter model: **PZEM-016** (100A external CT transformer) — suitable for whole-floor consumer unit monitoring
- User confirmed InfluxDB measurement naming: `235_floor_1` and `235_floor_2` (the "235" prefix likely refers to the physical circuit board or house address — treat as opaque identifiers)
- User wants the Power Profile chart to show all power flows: Production, Grid, Load, 235 Floor 1, 235 Floor 2 — giving a complete picture of where power is going per floor
- CT `energy` counter started at ~51 Wh when meters were installed today — it accumulates continuously and is **not** a daily reset counter (per PZEM datasheet: energy can only be reset via explicit software command)

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 08-update-house-load-measurement-to-use-ct-meters-on-both-consumer-units-add-per-floor-power-distribution-panels-and-add-data-reconciliation-panel-comparing-calculated-grid-solar-vs-actual-ct-measurement*
*Context gathered: 2026-04-02*
