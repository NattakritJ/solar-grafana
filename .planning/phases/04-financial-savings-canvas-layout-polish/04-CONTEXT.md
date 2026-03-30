# Phase 4: Financial Savings, Canvas Layout & Polish - Context

**Gathered:** 2026-03-30
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 4 delivers TOU-aware financial savings calculations (THB), a Canvas roof layout heatmap showing 8 panels in physical positions with color-coded production, a calendar heatmap for daily production patterns, and dashboard-wide visual consistency polish.

**In scope:** FINC-01, FINC-02, FINC-03, FINC-04, MODL-04, PROD-05, plus color/unit consistency audit
**Not in scope:** v2 requirements (VIS-01/02, ANLYT-01/02, ALRT-01), alerting rules, animated flows

</domain>

<decisions>
## Implementation Decisions

### TOU Savings Display
- **D-51:** Top-level savings presented as 3 stat panels in a row: Today's Savings (THB), Month-to-Date Savings (THB), Year-to-Date Savings (THB). Simple, scannable, consistent with overview stats row style from Phase 1.
- **D-52:** Peak vs off-peak breakdown (FINC-04) presented as 4 stat panels in a row: Peak kWh | Peak THB | Off-Peak kWh | Off-Peak THB. Compact, all values visible at a glance.
- **D-53:** Thai public holidays ARE classified as off-peak in TOU calculations. Hard-code 15-20 Thai public holiday dates per year. This requires annual maintenance of the holiday list but provides accurate TOU classification.
- **D-54:** THB currency formatted using Grafana `"none"` unit with `suffix: " THB"` (e.g., "145.50 THB"). Guaranteed to work across all themes and exports. No reliance on Grafana's built-in currency unit for THB.
- **D-55:** TOU rate classification in SQL must use `AT TIME ZONE 'Asia/Bangkok'` for all time extractions (hour, DOW). Peak = weekday (Mon-Fri, excluding holidays) 09:00-22:00 local time. Off-peak = all other times. Rates: Peak 5.7982 THB/kWh, Off-peak 2.6369 THB/kWh.
- **D-56:** Financial savings panels placed under "Financial Savings" row (id=600) — uncollapse it. Layout: top row = 3 summary stats (Today/MTD/YTD), bottom row = 4 breakdown stats (Peak kWh/THB + Off-Peak kWh/THB).

### Canvas Roof Layout
- **D-57:** 8 colored rectangles arranged in 2 rows with power value (W) overlaid as text on each rectangle. Color = production level using the established D-27/D-47 color scale (0W: #1a1a2e dark navy, 1-100W: #614a19 dark amber, 100-400W: #c7a035 amber, 400-600W: #73BF69 green, 600W+: #1a7c11 dark green).
- **D-58:** Panels labeled with logical names: "East PV1", "East PV2", "West PV1", etc. Row labels "West Slope" (top) and "East Slope" (bottom) on the side. Consistent with Phase 3 table panel naming (D-26).
- **D-59:** Canvas orientation — **West row on top, East row on bottom**. Custom left-to-right ordering:
  ```
  West (top):    [PV2] [PV1] [PV4] [PV3]
  East (bottom): [PV3] [PV4] [PV1] [PV2]
  ```
  Note: East row matches PROJECT.md physical layout. West row is mirrored (user-specified).
- **D-60:** Canvas panel placed under "Module Level" row (id=300) alongside existing bargauge and table panels, since it's a module-level visualization.

### Calendar Heatmap
- **D-61:** Use Grafana's built-in `heatmap` panel type with daily time buckets. Color gradient based on total daily kWh production. No external plugins needed.
- **D-62:** Color scheme uses the solar production scale: dark navy (0 kWh) to amber (mid) to dark green (high kWh). Consistent with D-27/D-47/D-57 production color scale across the dashboard.
- **D-63:** Default time range for the heatmap: last 90 days. Shows ~3 months of daily patterns for trend visibility. Can be overridden via Grafana time picker.
- **D-64:** Calendar heatmap placed under "Production" row (id=200) alongside existing production charts — it's a production visualization.

### Dashboard Polish (Agent's Discretion)
- **D-65:** Agent handles color/unit consistency audit across all phases — ensure all production panels use the same color scale, all power panels use consistent units (W vs kW), all tooltips have descriptive text.
- **D-66:** Agent handles panel spacing, row collapse defaults, and any minor tweaks to existing panels for visual consistency.
- **D-67:** Sunrise/sunset annotations deferred from Phase 1 — agent may include as a polish item if straightforward, otherwise defer to backlog.

### Agent's Discretion
- Exact Canvas panel dimensions and element positioning within the canvas
- Canvas element styling (border radius, padding, font size for W values)
- Whether savings stat panels include sparklines
- Panel IDs (continue from existing max, currently row IDs use 100-700, panel IDs go up to 37)
- Exact SQL implementation for TOU calculations (researcher validates AT TIME ZONE syntax)
- How to handle the 90-day default on heatmap (dashboard variable vs panel-level override)
- Extent of polish changes to existing Phase 1-3 panels

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project Specs
- `.planning/PROJECT.md` — TOU rates (Peak: 5.7982, Off-Peak: 2.6369), panel physical layout (PV3-PV4-PV1-PV2), InfluxDB schema, timezone (Asia/Bangkok)
- `.planning/REQUIREMENTS.md` — Phase 4 requirements: FINC-01-04, MODL-04, PROD-05 with acceptance criteria

### Research
- `.planning/research/STACK.md` — Canvas panel docs, heatmap panel type, SQL query patterns (DATE_BIN, CASE/WHEN for TOU), color scheme reference, currency formatting notes
- `.planning/research/PITFALLS.md` — Pitfall 3 (TZ mishandling for TOU — critical), Pitfall 13 (THB currency formatting), Pitfall 12 (PV position mapping for Canvas), Pitfall 8 (performance with long-range heatmap queries), Pitfall 11 (Canvas panel compatibility in Grafana 12)

### Prior Phase Context
- `.planning/phases/01-foundation-overview-stats/01-CONTEXT.md` — D-08 (__inputs pattern), D-13 (COALESCE), D-14 (color scheme foundation), D-15 (units)
- `.planning/phases/02-production-charts-grid-monitoring/02-CONTEXT.md` — D-17 (separate queries, no UNION ALL), D-20 (today_production counters for energy), D-23 (East=orange, West=blue)
- `.planning/phases/03-module-level-detail-inverter-health-event-log/03-CONTEXT.md` — D-26 (panel naming "East PV1"), D-27 (production color scale), D-47 (same color scale confirmed)

### Existing Dashboard
- `solar-pv-monitor.json` — Current dashboard with Phase 1-3 panels. Phase 4 adds panels to rows 200 (heatmap), 300 (Canvas), 600 (Financial). Panel IDs 38+.

### Roadmap
- `.planning/ROADMAP.md` SS "Phase 4" — Goal, requirements list, success criteria (5 criteria)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `stat` panel pattern from Phase 1 (overview KPIs) — reuse for financial savings summary stats
- `bargauge` and `table` panels from Phase 3 (module-level) — Canvas sits alongside these
- `barchart` panel from Phase 2 (Daily Energy) — heatmap complements this with different visualization
- `__inputs` datasource pattern (`${DS_INFLUXDB_SOLAR}`) — all new panels use this
- Production color scale (#1a1a2e through #1a7c11) — established in D-27, reuse for Canvas and heatmap

### Established Patterns
- Separate Grafana queries per measurement (no UNION ALL) — applies to TOU calculations querying both inverters
- `COALESCE(field, 0)` for nighttime NULLs on all power/energy fields
- `WHERE time >= $__timeFrom AND time <= $__timeTo` for time-range-aware queries
- `today_production_{N}_sensor` counters for daily energy (not power integration)
- `merge` + `reduce` transformations for combining multi-query stat panels

### Integration Points
- Dashboard JSON: add panels to rows 200 (heatmap), 300 (Canvas), 600 (Financial — currently collapsed, needs uncollapsing)
- Panel IDs continue from 38+ (max non-row ID is 37)
- Row IDs already exist: 200 (Production), 300 (Module Level), 600 (Financial Savings)
- TOU SQL queries will need `AT TIME ZONE 'Asia/Bangkok'` — needs runtime validation per STATE.md concern
- Canvas panel JSON structure may need building in Grafana UI first then extracting — per STATE.md concern

</code_context>

<specifics>
## Specific Ideas

- User specified a custom Canvas layout ordering that differs from PROJECT.md physical layout: West row is mirrored (PV2-PV1-PV4-PV3) while East row keeps physical order (PV3-PV4-PV1-PV2). This is an intentional user choice — do not "correct" it to match PROJECT.md.
- Thai public holidays must be hard-coded in the TOU SQL. Researcher should compile the standard Thai public holiday list for 2026 (and ideally make it easy to update annually).
- The financial savings row (id=600) is currently the only collapsed row — it needs to be uncollapsed and populated.
- Dashboard polish scope was explicitly left to agent's discretion (not selected for discussion). Keep polish lightweight — consistency audit, not redesign.

</specifics>

<deferred>
## Deferred Ideas

- Sunrise/sunset Grafana annotations — low priority, agent may include as polish if trivial
- Animated energy flow diagram (VIS-01) — v2 requirement
- CO2 offset estimation (VIS-02) — v2 requirement
- System efficiency gauge (ANLYT-01) — v2 requirement
- Grafana alerting rules (ALRT-01) — separate concern from dashboard JSON

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 04-financial-savings-canvas-layout-polish*
*Context gathered: 2026-03-30*
