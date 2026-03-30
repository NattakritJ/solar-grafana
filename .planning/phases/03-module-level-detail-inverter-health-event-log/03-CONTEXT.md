# Phase 3: Module-Level Detail, Inverter Health & Event Log - Context

**Gathered:** 2026-03-30
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 3 adds per-panel module-level monitoring (power, voltage, current for all 8 panels), inverter health tracking (temperature, device state, alarms, faults), and event logging (alarm/fault history, grid backfeed detection, unified event log) to the existing dashboard.

**In scope:** MODL-01, MODL-02, MODL-03, HLTH-01, HLTH-02, HLTH-03, HLTH-04, EVNT-01, EVNT-02, EVNT-03
**Not in scope:** MODL-04 (Canvas roof layout heatmap → Phase 4), PROD-05 (calendar heatmap → Phase 4), FINC-01–04 (financial → Phase 4)

</domain>

<decisions>
## Implementation Decisions

### Module-Level Panel Layout
- **D-24:** Use a `table` panel for the primary module-level view — one row per panel (8 rows), columns for: Panel Label, Current Power (W), Voltage (V), Current (A), Today's kWh, Total kWh. This is the most information-dense format and allows cell-level color coding for underperforming panels.
- **D-25:** Use a `bargauge` panel showing current power (W) for all 8 panels as horizontal bars — provides instant visual comparison of which panels are producing more/less. Group by inverter: East PV1-PV4 first, then West PV1-PV4.
- **D-26:** Panel labels in both table and bar gauge use descriptive names: "East PV1", "East PV2", etc. (not just "PV1") to avoid ambiguity.
- **D-27:** Table panel uses cell-level color thresholds on the Power column: 0W = dark-gray, 1-100W = dark-amber, 100-400W = amber, 400-600W = green, 600W+ = dark-green. Same color scale as planned for Canvas roof layout in Phase 4 (consistent visual language).
- **D-28:** Today's and total production per panel use `today_production_{N}_sensor` and `total_production_{N}_sensor` fields from each inverter measurement. These are the inverter's built-in counters — consistent with the daily energy approach from Phase 2 (D-20).

### Nighttime & Offline Handling
- **D-29:** When inverters are offline at night, stat/gauge panels show "0 W" with dark-gray coloring — NOT "No Data" or error states. Use `COALESCE(field, 0)` in all queries (consistent with D-13).
- **D-30:** Inverter device state uses value mapping: state values mapped to human-readable labels with color coding. Normal/producing = green, Standby = yellow, Offline/sleeping = gray (not red). HLTH-04 requirement: nighttime offline is visually neutral, not alarming.
- **D-31:** Temperature gauges at night should show last known value (via `ORDER BY time DESC LIMIT 1` with no time range filter on the stat query) or 0 if no data exists. Nighttime temperature is not alarming.

### Inverter Health Display
- **D-32:** Use `gauge` panels for inverter temperature — one gauge per inverter (East, West). Thresholds: 0-40°C = green, 40-60°C = yellow, 60°C+ = red. This provides immediate visual severity.
- **D-33:** Use `stat` panels for device state, alarm status, and fault status — 3 stats per inverter (6 total), arranged as a compact row. Each uses value mapping for color: Normal/0 = green, Active alarm/fault = red.
- **D-34:** Device state values (from `device_state_sensor`) mapped to labels: 0="Offline", 1="Standby", 2="Producing", etc. Exact mapping depends on inverter manufacturer's state codes — researcher should verify the actual enum values from the live data.
- **D-35:** Alarm and fault sensors (`alarm_sensor`, `fault_sensor`) — 0 = no alarm/fault (green), any non-zero value = active (red). Display the raw code alongside the color for diagnostic value.

### Alarm/Fault History & Event Log
- **D-36:** Use a `table` panel for alarm/fault history log showing: Timestamp, Inverter (East/West), Type (Alarm/Fault/State Change), Code, Description. Query uses `WHERE alarm_sensor != 0 OR fault_sensor != 0 OR device_state_sensor changed` logic. Sorted by time descending (most recent first).
- **D-37:** Use a `state-timeline` panel for device state history — shows producing/standby/offline transitions over time as colored horizontal bars. One lane per inverter. This fulfills the "when were inverters producing vs idle" use case from STACK.md.

### Grid Backfeed Event Tracking
- **D-38:** Grid backfeed defined as `power_sensor < 0` from the "Smart Meter" measurement. Each contiguous period of negative power = one backfeed event.
- **D-39:** Use 2 `stat` panels for backfeed summary: "Today's Backfeed Count" and "Max Backfeed Power (W)". These sit at the top of the Event Log section for at-a-glance monitoring.
- **D-40:** Use a `table` panel for the backfeed event log: Timestamp, Power (W, absolute value), Duration. Duration calculation uses time difference between first and last negative reading in a contiguous group — researcher should evaluate whether this is feasible in a single SQL query or needs Grafana transformations.
- **D-41:** Backfeed power displayed as absolute value (positive number) with a descriptive column header like "Export Power (W)" to avoid confusion with negative numbers in the UI.

### Unified Event Log
- **D-42:** EVNT-03 unified event log combines: backfeed events, alarm events, fault events, and device state changes in one `table` panel, sorted chronologically (newest first). Columns: Timestamp, Source (Smart Meter / East Inverter / West Inverter), Event Type (Backfeed / Alarm / Fault / State Change), Detail.
- **D-43:** This likely requires UNION ALL across multiple subqueries — which InfluxDB 3 Core does NOT support in subquery form (per D-17/STATE.md bug fix). Approach: use separate Grafana queries (one per event type) with `merge` transformation to combine into a single table. Researcher should validate this approach works for table panels with different column schemas.

### Row Placement & Dashboard Structure
- **D-44:** Module-level panels go under "Module Level" row (id=300) — uncollapse it.
- **D-45:** Inverter health panels (temperature, state, alarms) go under "Inverter Health" row (id=500) — uncollapse it.
- **D-46:** Event log panels (backfeed stats, backfeed log, alarm log, state timeline, unified log) go under "Event Log" row (id=700) — uncollapse it.

### Color Scheme
- **D-47:** Module-level power follows the production color scale: dark-gray → dark-amber → amber → green → dark-green (0W → 600W+). Consistent with Phase 4 Canvas heatmap colors.
- **D-48:** East panels use orange-tinted labels, West panels use blue-tinted labels — consistent with D-23 (Phase 2 East=orange, West=blue).
- **D-49:** Temperature: green → yellow → red gradient (cool → warm → hot).
- **D-50:** Alarm/fault indicators: green = normal, red = active. State changes: green = producing, yellow = standby, gray = offline.

### Agent's Discretion
- Exact panel widths/heights within the 24-column grid
- Panel ordering within rows (as long as module-level data comes first, then health, then events)
- Whether to include sparklines on module-level stat panels
- Exact `gridPos` y-coordinates for new rows
- Panel IDs (keep sequential from Phase 2's last ID, currently id=21)
- Whether alarm/fault log includes a time-range filter or shows all-time history
- Exact SQL for backfeed duration calculation (single query vs transformation)
- Whether state-timeline panel is included (nice-to-have) or only the table log (must-have)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project Specs
- `.planning/PROJECT.md` — Hardware specs (8 panels, 2 inverters), InfluxDB schema (pv1-4 fields, device_state/alarm/fault sensors), panel physical layout (PV3-PV4-PV1-PV2 order)
- `.planning/REQUIREMENTS.md` — Phase 3 requirements: MODL-01–03, HLTH-01–04, EVNT-01–03 with acceptance criteria

### Research
- `.planning/research/STACK.md` — Panel types (table, bargauge, gauge, stat, state-timeline), SQL query patterns (LAG, DATE_BIN), color scheme, units reference
- `.planning/research/PITFALLS.md` — Pitfall 9 (nighttime NULLs), Pitfall 12 (per-panel aggregation vs total), Pitfall 1 (SQL syntax), Pitfall 6 (quoted measurement names)

### Prior Phase Context
- `.planning/phases/01-foundation-overview-stats/01-CONTEXT.md` — D-08 (__inputs pattern), D-12 (quoted names), D-13 (COALESCE), D-09 (latest-value query pattern)
- `.planning/phases/02-production-charts-grid-monitoring/02-CONTEXT.md` — D-17 (separate queries, no UNION ALL), D-20 (today_production counters), D-22 (row placement), D-23 (East/West color scheme)

### Existing Dashboard
- `solar-pv-monitor.json` — Current dashboard with Phase 1+2 panels. Phase 3 adds panels to rows 300 (Module Level), 500 (Inverter Health), 700 (Event Log). Last panel id=21.

### Roadmap
- `.planning/ROADMAP.md` §"Phase 3" — Goal, requirements list, success criteria (7 criteria)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `bargauge` panel pattern from Phase 2 (id=13, Inverter Power) — reuse for per-panel power comparison
- `stat` panel pattern from Phase 1 (8 overview stats) — reuse for health indicators and backfeed summary stats
- `timeseries` panel pattern from Phase 2 — reuse for state-timeline if needed
- `__inputs` datasource pattern established in Phase 1 — all new panels use `${DS_INFLUXDB_SOLAR}`

### Established Patterns
- Separate Grafana queries per measurement (no UNION ALL) — critical for unified event log approach
- `merge` + `reduce` transformations for combining multi-query stat panels (Phase 1 bug fix)
- `COALESCE(field, 0)` for nighttime NULLs on all power fields
- `WHERE time >= $__timeFrom AND time <= $__timeTo` for time-range-aware queries (Phase 2)
- `ORDER BY time DESC LIMIT 1` for latest-value queries (Phase 1)

### Integration Points
- Dashboard JSON: add panels to existing rows 300, 500, 700 (currently collapsed and empty)
- Panel IDs continue from 22+ (Phase 2 ends at id=21)
- New panels follow same `gridPos` system (24-column grid)
- Queries use same InfluxDB measurements: "East Microinverter", "West Microinverter", "Smart Meter"

</code_context>

<specifics>
## Specific Ideas

- User explicitly delegated all 4 gray areas to agent discretion ("You decide everything").
- Key constraint from Pitfall 12: per-panel data uses `pv{N}_power/voltage/current_sensor` fields, NOT `power_sensor` (which is inverter total after losses).
- Physical panel layout (PV3-PV4-PV1-PV2 left to right) matters for Phase 4 Canvas but should also be documented in Phase 3 table column ordering for consistency.
- HLTH-04 is a critical UX requirement: nighttime offline must look normal, not alarming. This affects value mappings, thresholds, and color choices across all health panels.
- Backfeed event detection (power_sensor < 0) on Smart Meter is a novel query pattern not used in Phase 1-2 — researcher should validate this works and what the actual negative values look like in the data.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 03-module-level-detail-inverter-health-event-log*
*Context gathered: 2026-03-30*
