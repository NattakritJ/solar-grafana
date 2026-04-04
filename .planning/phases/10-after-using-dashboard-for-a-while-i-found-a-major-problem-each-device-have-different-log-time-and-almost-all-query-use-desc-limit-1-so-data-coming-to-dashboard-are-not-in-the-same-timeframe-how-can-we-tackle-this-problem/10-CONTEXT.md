# Phase 10: Temporal Alignment — Replace DESC LIMIT 1 / selector_last with AVG/MAX - Context

**Gathered:** 2026-04-04
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 10 fixes a cross-device temporal alignment problem: all "real-time snapshot" panels (stat, bargauge, canvas) currently use `selector_last(field, time)['value']` or `ORDER BY time DESC LIMIT 1` to fetch the most recent reading from each device. Since each device has a different log timestamp (and some log 5× faster than others), queries firing at the same time return values from meaningfully different instants — causing cross-device expression panels like Self-Consumption to combine non-contemporaneous readings.

The fix is to replace point-in-time latest-value queries with `AVG(field)` over the existing 2-minute recency window, so all devices report their average over the same 2-minute timeframe. Cumulative production counters use `MAX(field)` instead (they monotonically increase — MAX = most recent/highest). Categorical fields (state, alarm, fault) are kept on `selector_last` unchanged.

**In scope:**
- Replace `selector_last(power/voltage/current/temperature field, time)['value']` with `AVG(field)` across all stat/bargauge/canvas panels
- Replace `ORDER BY time DESC LIMIT 1` with `AVG(field)` on the 6 grid/Smart Meter latest-value queries
- Replace `selector_last(today_production_N_sensor, time)['value']` with `MAX(today_production_N_sensor)` (8 occurrences)
- Replace `selector_last(total_production_N_sensor, time)['value']` with `MAX(total_production_N_sensor)` (8 occurrences)
- All changes within `solar-pv-monitor.json` only — no new panels, no gridPos changes

**Out of scope:**
- `selector_last(device_state_sensor, time)['value']` — KEEP (categorical, not numeric)
- `selector_last(device_alarm_sensor, time)['value']` — KEEP (categorical)
- `selector_last(device_fault_sensor, time)['value']` — KEEP (categorical)
- Time-series panels with `DATE_BIN` — unaffected (already aggregate over time buckets)
- Today/lifetime energy panels with `date_trunc` + `MAX` — already correct, unchanged
- Panel 702 (Lifetime Free Energy) — already uses `MAX(total_production_*)` with no time filter, already correct

</domain>

<decisions>
## Implementation Decisions

### Fix Strategy

- **D-01:** Use `AVG(field)` over the same 2-minute recency window as the replacement for both:
  - `COALESCE(selector_last(field, time)['value'], 0)` patterns
  - `SELECT field FROM table WHERE time >= now() - INTERVAL '2 minutes' ORDER BY time DESC LIMIT 1` patterns

  New form for `selector_last` targets:
  `COALESCE(AVG(field), 0) AS alias`

  New form for `LIMIT 1` targets:
  `SELECT AVG(field) AS alias FROM table WHERE time >= now() - INTERVAL '2 minutes'`
  (Remove `ORDER BY time DESC LIMIT 1`; wrap field in `AVG()`)

- **D-02:** Time window stays at `2 minutes` — consistent with Phase 7 recency window decision. No change to the window duration.

### Field-Type Rules

- **D-03:** Power / Voltage / Current / Temperature fields → `AVG(field)`
  Covers: `power`, `power_sensor`, `pv{1-4}_{power,voltage,current}_sensor`, `voltage_sensor`, `current_sensor`, `temperature_sensor`, `voltage`, `current`, `frequency_sensor`, `power_factor_sensor`
  Rationale: These are instantaneous measurements — averaging 2-min readings aligns all devices to the same temporal window and smooths high-frequency CT readings (2s interval) to match 10s-interval inverters.

- **D-04:** Cumulative production counter fields → `MAX(field)`
  Covers: `today_production_{1,2,3,4}_sensor`, `total_production_{1,2,3,4}_sensor`
  Rationale: These are monotonically increasing counters (they only go up). `AVG` over 2 minutes would return a value slightly below current. `MAX` = the most recent (highest) reading = correct current value.

- **D-05:** Categorical state/alarm/fault fields → **KEEP** `selector_last` unchanged
  Covers: `device_state_sensor`, `device_alarm_sensor`, `device_fault_sensor`
  Panels: 8 (System Status), 26 (East State), 27 (East Alarm), 28 (East Fault), 29 (West State), 30 (West Alarm), 31 (West Fault)
  Rationale: These are string/enum values — `AVG` is meaningless on them.

### Module Detail Table (Panel 23)

- **D-06:** Panel 23 (Module Detail) has 6 columns per PV channel query:
  - Power (W) → `AVG(pv{N}_power_sensor)`
  - Voltage (V) → `AVG(pv{N}_voltage_sensor)`
  - Current (A) → `AVG(pv{N}_current_sensor)`
  - Today (kWh) → `MAX(today_production_{N}_sensor)`
  - Total (kWh) → `MAX(total_production_{N}_sensor)`
  Both rules apply within the same 8-target query set.

### Replacement Scope Summary

| Change type | Pattern | Count | New form |
|------------|---------|-------|----------|
| selector_last power/VC/temp | `selector_last(field, time)['value']` | 70 | `AVG(field)` |
| selector_last today_production | `selector_last(today_production_N_sensor, time)['value']` | 8 | `MAX(today_production_N_sensor)` |
| selector_last total_production | `selector_last(total_production_N_sensor, time)['value']` | 8 | `MAX(total_production_N_sensor)` |
| ORDER BY DESC LIMIT 1 | 6 unique SQL strings | 6 queries | AVG-based SELECT |
| **state/alarm/fault kept** | `selector_last(device_*)` | 8 | **unchanged** |
| **Total changes** | | **~92** | |

### Agent's Discretion

- Exact sequencing of replaceAll operations within the plan (e.g., whether to do production counters first or last)
- Whether to use a single regex-based replaceAll for the `selector_last(...)['value']` pattern or individual per-field replacements
- How to handle the `power_factor_sensor` COALESCE fallback value (`0` vs keeping current) — use the same `0` default unless existing query differs

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
- `.planning/phases/07-fix-stale-data-when-inverter-goes-offline-at-sunset/07-01-PLAN.md` — Phase 7 established the 2-minute recency window (`WHERE time >= now() - INTERVAL '2 minutes'`). Phase 10 keeps this window — only the aggregation function changes.
- `.planning/phases/08-update-house-load-measurement-to-use-ct-meters-on-both-consumer-units-add-per-floor-power-distribution-panels-and-data-reconciliation-panel-comparing-calculated-grid-solar-vs-actual-ct-measurement/08-CONTEXT.md` — CT meter field naming (no `_sensor` suffix for `power`, `voltage`, `current`)
- `.planning/phases/09-update-panels-to-use-new-ct-meter-data-for-grid-energy-import-current-voltage-real-power-instead-of-smart-meter/09-CONTEXT.md` — Confirms `grid` table schema; Smart Meter now used only for frequency/power_factor

### InfluxDB Live Schema (verified 2026-04-04)

Device logging characteristics confirmed from live data:

| Device | Log Interval | Pattern | Max Skew vs. Now |
|--------|-------------|---------|-----------------|
| East Microinverter | ~10s | :08, :18, :28... | 0–10s |
| West Microinverter | ~10s | :07, :17, :27... | 0–10s |
| Smart Meter | ~10s | :09, :19, :29... | 0–10s |
| grid CT | ~2s | every ~2 seconds | 0–2s |
| 235_floor_1 | ~2s | every ~2 seconds | 0–2s |
| 235_floor_2 | ~2s | every ~2 seconds | 0–2s |

Key finding: East/West/Smart Meter are ~1s apart (same 10s interval); CT meters are 5× faster. With `ORDER BY time DESC LIMIT 1`, a cross-device expression (e.g., Self-Consumption = Solar/[Solar+Grid]) can combine an inverter reading up to 10s old with a CT reading that's 1s old — a 9-second mismatch. `AVG` over 2 minutes eliminates this by aligning all devices to the same temporal window.

</canonical_refs>

<code_context>
## Existing Code Insights

### Query Patterns Currently in Dashboard

**selector_last pattern (most common — 86 occurrences):**
```sql
SELECT COALESCE(selector_last(power_sensor, time)['value'], 0) AS solar_kw
FROM "East Microinverter"
WHERE time >= now() - INTERVAL '2 minutes'
```
→ Replace `selector_last(power_sensor, time)['value']` with `AVG(power_sensor)`

**ORDER BY DESC LIMIT 1 pattern (6 occurrences):**
```sql
SELECT power AS grid_kw
FROM "grid"
WHERE time >= now() - INTERVAL '2 minutes'
ORDER BY time DESC LIMIT 1
```
→ Change to: `SELECT AVG(power) AS grid_kw FROM "grid" WHERE time >= now() - INTERVAL '2 minutes'`

**Production counter pattern (unchanged target form, just replace function):**
```sql
COALESCE(selector_last(today_production_1_sensor, time)['value'], 0) AS "Today (kWh)"
```
→ `COALESCE(MAX(today_production_1_sensor), 0) AS "Today (kWh)"`

### Panels Needing Changes

| Panel | Title | Changes needed |
|-------|-------|----------------|
| 1 | Solar Power | 2 selector_last → AVG (hidden targets A, B) |
| 4 | Grid Import | 1 LIMIT 1 → AVG |
| 5 | House Load | 2 selector_last → AVG (hidden Floor1, Floor2) |
| 6 | Self-Consumption | 3 selector_last → AVG (hidden East, West, Grid) |
| 9 | ☀️ Solar → House | 2 selector_last → AVG (hidden A, B) |
| 10 | 🏠 House Load | 2 selector_last → AVG (hidden Floor1, Floor2) |
| 11 | House ← Grid ⚡ | 1 LIMIT 1 → AVG |
| 13 | Inverter Power | 2 selector_last → AVG (East, West) |
| 16 | Grid Voltage | 1 LIMIT 1 → AVG |
| 17 | Grid Frequency | 1 LIMIT 1 → AVG (Smart Meter frequency_sensor) |
| 18 | Power Factor | 1 LIMIT 1 → AVG (Smart Meter power_factor_sensor) |
| 19 | Grid Current | 1 LIMIT 1 → AVG |
| 22 | Panel Power | 8 selector_last → AVG (all 8 PV channels) |
| 23 | Module Detail | 24 selector_last (8 channels × {Power→AVG, Voltage→AVG, Current→AVG, Today→MAX, Total→MAX}) |
| 24 | East Temperature | 1 selector_last → AVG |
| 25 | West Temperature | 1 selector_last → AVG |
| 45 | Roof Layout | 8 selector_last → AVG (pv{1-4}_power_sensor for each inverter) |
| 902 | 🏠 Floor 1 Power | 1 selector_last → AVG |
| 903 | 🏠 Floor 2 Power | 1 selector_last → AVG |
| 904 | ⚡ CT Load | 2 selector_last → AVG (hidden Floor1, Floor2) |
| 905 | 🔢 Calculated Load | 3 selector_last → AVG (hidden East, West, Grid) |

### Panels NOT Changing (kept as-is)

| Panel | Reason |
|-------|--------|
| 8 (System Status) | Uses selector_last for `device_state_sensor` — categorical, must keep |
| 26–31 (State/Alarm/Fault stats) | All use selector_last for device_state/alarm/fault — categorical |
| 2 (Today's Free Energy) | Uses MAX(today_production_*) with date_trunc boundary — already correct |
| 7 (Peak Today) | Uses MAX(power_sensor) over today — already correct |
| 702 (Lifetime Free Energy) | Uses MAX(total_production_*) with no time filter — already correct |
| 38/39/40 (Savings) | Use DATE_BIN + SUM over time ranges — unaffected |
| 12/14/15/20/21/801/46 | Time-series with DATE_BIN — unaffected |

### Established Patterns
- All `COALESCE(..., 0)` wrappers stay — they correctly handle the no-data case at night
- The `WHERE time >= now() - INTERVAL '2 minutes'` window stays on every affected query
- Expression (math) targets in panels 1, 5, 6, 9, 10, 904, 905 etc. don't change — they operate on the AVG outputs

### Integration Points
- All changes are in-place SQL replacements in `rawSql` strings
- No new panels, no row additions, no gridPos changes
- Expression math targets are unaffected (they reference refIds whose underlying SQL changes)
- The 2-minute `now()` window ensures night-time "no data" behavior is preserved (when inverters are offline, no rows → AVG returns NULL → COALESCE returns 0)

</code_context>

<specifics>
## Specific Ideas

- Live data confirmed East Microinverter output was stable at 58.8W for 5+ consecutive minutes — this confirms AVG over 2 minutes gives the same result as LIMIT 1 for typical inverter operation
- Grid CT at 2s interval: AVG over 30s window gave 1292.2W vs LIMIT 1 gave 1289.7W — negligible difference (2.5W / 0.2%) but better temporal alignment
- `power_factor_sensor` from Smart Meter and `frequency_sensor` from Smart Meter are both standalone panels (no cross-device expression) — the AVG fix still improves them because they go from a reading up to 10s old to an average over 2 minutes, which is smoother

</specifics>

<deferred>
## Deferred Ideas

- **grid.energy_today for Today's Grid Import stat** — A new stat panel showing daily kWh imported from grid using `grid.energy_today` counter field. Deferred from Phase 9. Still out of scope for Phase 10.
- **air_conditioner monitoring** — `air_conditioner` table exists with power/voltage/current. Power Distribution row enhancement. Separate phase.
- **Tighter 30s window option** — If the homeowner ever wants faster "right now" response (e.g., for a wall display), AVG over 30s could be explored. At 30s window, inverters have 3 readings (adequate), CT meters have 15 readings (good). Deferred — the 2-minute window from Phase 7 is the established preference.

</deferred>

---

*Phase: 10-after-using-dashboard-for-a-while-i-found-a-major-problem-each-device-have-different-log-time-and-almost-all-query-use-desc-limit-1-so-data-coming-to-dashboard-are-not-in-the-same-timeframe-how-can-we-tackle-this-problem*
*Context gathered: 2026-04-04*
