# Phase 1: Foundation & Overview Stats - Context

**Gathered:** 2026-03-30
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 1 delivers an importable Grafana 12.4.1 dashboard JSON skeleton with:
- All structural scaffolding (rows, dashboard metadata, datasource `__inputs`)
- 9 overview stat panels showing real-time solar production, grid import, house load, energy totals, self-consumption ratio, system status, peak power today
- Power flow visualization (Solar → House ← Grid)
- All queries using valid InfluxDB 3 Core SQL syntax

**In scope:** INFR-01 through INFR-06, OVER-01 through OVER-09.
**Not in scope:** Production charts, module-level panels, financial calculations, or Canvas roof layout — those are Phases 2–4.

</domain>

<decisions>
## Implementation Decisions

### Stat Panel Layout & Density
- **D-01:** Agent's discretion — use a single compact row of stat panels (full 24-column width) for the overview KPIs. Suggested arrangement: Solar Power | Today's kWh | Lifetime kWh | Grid Import | House Load | Self-Consumption % | Peak Today | System Status. Each stat panel transparent background, no borders, tight spacing. Width: approximately `w:3` for 8 panels.
- **D-02:** All overview stats use `"reduceOptions": {"calcs": ["lastNotNull"]}` — always show the most recent value, never a time-range aggregate.
- **D-03:** Self-consumption ratio (OVER-06) calculated in SQL as `solar_power / (solar_power + grid_import) * 100`, clamped to 0–100.

### Power Flow Visualization
- **D-04:** Agent's discretion — implement OVER-08 as three stat panels arranged left-to-right (`Solar Production` | `House Load` | `Grid Import`) with descriptive panel titles that communicate direction. No canvas or text-arrow panels needed for Phase 1. Keep it simple and readable.
- **D-05:** Power flow row uses `"transparent": true` on all panels so they appear as a unified bar. Separate from the 9-stat KPI row above.

### Dashboard Scaffolding
- **D-06:** Phase 1 creates the **full dashboard skeleton** with all collapsible row panels for all 4 phases, even if later rows are empty. This avoids JSON restructuring in later phases.
  - Row 1: "Overview" (collapsed: false — visible by default)
  - Row 2: "Production" (collapsed: true — placeholder)
  - Row 3: "Module Level" (collapsed: true — placeholder)
  - Row 4: "Grid & Consumption" (collapsed: true — placeholder)
  - Row 5: "Inverter Health" (collapsed: true — placeholder)
  - Row 6: "Financial Savings" (collapsed: true — placeholder)
  - Row 7: "Event Log" (collapsed: true — placeholder)
- **D-07:** Dashboard-level settings: `"title": "Solar PV Monitor"`, `"timezone": "Asia/Bangkok"`, `"refresh": "10s"`, `"time": {"from": "now-24h", "to": "now"}`, `"schemaVersion": 40`.
- **D-08:** Datasource referenced via `__inputs` pattern: `"${DS_INFLUXDB_SOLAR}"` — required for portability on import (avoids UID mismatch, see PITFALLS.md Pitfall 4).

### SQL Query Strategy
- **D-09:** All Phase 1 stat panels use **"latest value" queries** — `WHERE time >= now() - INTERVAL '1 minute' ORDER BY time DESC LIMIT 1`. This is the safest approach for Phase 1 since it avoids $__timeFrom/$__timeTo macro validation risk (the #1 flagged risk in STATE.md).
- **D-10:** Time-range macros (`$__timeFrom`, `$__timeTo`) are **deferred to Phase 2** where they are required for production charts. Phase 1 validates basic SQL connectivity, not macro substitution.
- **D-11:** Total system power query sums `power_sensor` from BOTH `"East Microinverter"` AND `"West Microinverter"` measurements (NOT pv channel sums — see PITFALLS.md Pitfall 12). Grid import from `"Smart Meter"` `power_sensor`.
- **D-12:** All measurement names double-quoted in SQL: `"East Microinverter"`, `"West Microinverter"`, `"Smart Meter"` (PITFALLS.md Pitfall 6).
- **D-13:** `COALESCE(power_sensor, 0)` applied to all power fields to handle nighttime NULLs (PITFALLS.md Pitfall 9).

### Color Scheme
- **D-14:** Agent's discretion — follow the color palette from STACK.md:
  - Solar production: yellow → green gradient thresholds (0W: dark-gray, 1W: yellow, 1000W: green, 4000W: dark-green)
  - Grid import: blue
  - House load: orange
  - Self-consumption %: green gradient (0%: dark-gray, 30%: yellow, 70%: green, 100%: dark-green)
  - System status: value mapping (Normal: green, Fault: red, Offline: dark-gray)

### Units
- **D-15:** Overview stats use `"kwatt"` (kW) for power values, `"kwatth"` (kWh) for energy, `"percentunit"` for self-consumption ratio. All panels include descriptive panel descriptions with unit labels.

### the Agent's Discretion
- Exact panel widths/heights within the 24-column grid
- Panel ordering within rows (as long as it's logical and follows OVER-01 through OVER-09 order)
- Whether to include a sparkline on any stat panels (graphMode: "area" vs "none")
- Panel IDs and refIds (just keep them unique and sequential)
- Whether "system status" uses `device_state_sensor` from one or both inverters (use East as primary, West as secondary check)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project Specs
- `.planning/PROJECT.md` — Hardware specs, panel layout, InfluxDB schema, TOU rates, constraints
- `.planning/REQUIREMENTS.md` — Full requirement list for Phase 1: INFR-01–06, OVER-01–09 with acceptance criteria

### Research
- `.planning/research/STACK.md` — Grafana panel types, SQL query patterns, Dashboard JSON structure, color scheme, units reference
- `.planning/research/PITFALLS.md` — Critical pitfalls: SQL syntax traps (Pitfall 1), measurement names (Pitfall 6), datasource UID (Pitfall 4), nighttime NULLs (Pitfall 9), macro substitution (Pitfall 10)

### Phase Roadmap
- `.planning/ROADMAP.md` §"Phase 1: Foundation & Overview Stats" — Goal, requirements list, success criteria

### No external ADRs or specs — requirements fully captured in decisions and research files above.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- None — this is a greenfield project. No existing dashboard JSON, components, or utilities to reuse.

### Established Patterns
- None yet — Phase 1 establishes all patterns. The `__inputs` datasource pattern, `rawSql` query format, and color threshold structure defined here will be the baseline for all subsequent phases.

### Integration Points
- The dashboard JSON file created in Phase 1 is the single artifact. All subsequent phases ADD panels/rows to this JSON file.
- Phase 2 will add time-series panels using `$__timeFrom`/`$__timeTo` macros — the Phase 1 JSON structure must accommodate this by using the correct `schemaVersion: 40` and datasource variable pattern.

</code_context>

<specifics>
## Specific Ideas

- User explicitly delegated all 4 gray areas to agent discretion ("Let you decide all").
- STACK.md research recommends `"transparent": true` for the overview stats row to create a clean KPI bar — adopt this.
- PITFALLS.md strongly recommends validating datasource connectivity BEFORE building panels — Phase 1 should include a simple "test query" stat panel as the first panel built, before adding all 9 overview stats.
- The `selector_last(field, time)['value']` InfluxDB selector function is an alternative to `ORDER BY time DESC LIMIT 1` for latest-value queries — researcher should evaluate which is more reliable for InfluxDB 3 Core.

</specifics>

<deferred>
## Deferred Ideas

- Time-range macro validation (`$__timeFrom`/`$__timeTo`) — deferred to Phase 2 where time-series charts require it
- Dynamic bin-size scaling with `$__interval` — deferred to Phase 2
- Grafana annotations (sunrise/sunset) — deferred to Phase 4 polish
- `"graphMode": "area"` sparklines on stat panels — agent can include or skip, low priority

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 01-foundation-overview-stats*
*Context gathered: 2026-03-30*
