# Phase 5: Fix Transformation-Based Calculations → Expressions - Context

**Gathered:** 2026-04-01
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 5 migrates all panels that use Grafana client-side transformations **for calculation** to use server-side Grafana Expression targets (type=math) instead. The goal is to remove `calculateField`, `reduce(sum)`, and blank Expression targets — replacing them with explicit `$A + $B` style math targets.

**In scope:** Panels where transformations perform arithmetic (sum, ratio, difference) on multi-query data
**Out of scope:** Panels that use `merge` only for combining queries for display (table, bargauge, canvas, barchart, piechart) — `merge` is acceptable for display purposes

</domain>

<decisions>
## Implementation Decisions

### Scope — Which Panels to Migrate

- **D-01:** Migrate only panels where transformations are used **for calculation**. Panels using `merge` purely for display (combining multiple series into one panel) are out of scope for this phase.

- **D-02:** **In scope — 14 panels total:**

  | Panel | Title | Current transforms | Migration action |
  |-------|-------|--------------------|-----------------|
  | 5 | House Load | merge + blank math expr `Total` | Fill `Total.expr = $East + $West + $Grid`, remove merge |
  | 6 | Self-Consumption | merge (disabled) + 3×calculateField (disabled) + blank math exprs Solar/Total/Self | Fill expressions, remove all disabled transforms |
  | 10 | 🏠 House Load | merge + blank math expr `Total` | Same as panel 5 |
  | 1 | Solar Power | merge + reduce(sum) | Replace with Expression: `$A + $B` |
  | 2 | Today's Energy | merge + reduce(sum) | Replace with Expression: `$A + $B` |
  | 3 | Lifetime Energy | merge + reduce(sum) | Replace with Expression: `$A + $B` |
  | 7 | Peak Today | merge + reduce(sum) | Replace with Expression: `$A + $B` |
  | 9 | ☀ Solar → House | merge + reduce(sum) | Replace with Expression: `$A + $B` |
  | 38 | Today's Savings | merge + reduce(sum) | Replace with Expression: `$A + $B` |
  | 39 | Month Savings | merge + reduce(sum) | Replace with Expression: `$A + $B` |
  | 40 | Year Savings | merge + reduce(sum) | Replace with Expression: `$A + $B` |
  | 41 | Peak kWh | merge + reduce(sum) | Replace with Expression: `$A + $B` |
  | 42 | Peak THB | merge + reduce(sum) | Replace with Expression: `$A + $B` |
  | 43 | Off-Peak kWh | merge + reduce(sum) | Replace with Expression: `$A + $B` |
  | 44 | Off-Peak THB | merge + reduce(sum) | Replace with Expression: `$A + $B` |

- **D-03:** **Out of scope — keep merge as-is:**
  - Panel 13 (Inverter Power bargauge) — merge only for display
  - Panel 14 (East/West Piechart) — merge only for display
  - Panel 15 (Daily Energy barchart) — merge only for display
  - Panel 22 (Panel Power bargauge) — merge only for display
  - Panel 23 (Module Detail table) — merge only for display
  - Panel 45 (Roof Layout canvas) — merge only for display

### Expression Formulas

- **D-04:** **Panel 5 & 10 (House Load):** The Expression target `Total` should use `expr = $East + $West + $Grid`. This sums the kW values from East Microinverter, West Microinverter, and Smart Meter. Rename `Total` to the display field via `reduceOptions`. Remove the `merge` transformation after adding the Expression.

- **D-05:** **Panel 6 (Self-Consumption):** Three Expression targets:
  - `Solar.expr = $East + $West` (total solar production in W)
  - `Total.expr = $Solar + $Grid` (total house load in W)
  - `Self.expr = $Solar / $Total` (self-consumption ratio, 0–1 → displays as % via `percentunit`)
  Remove all disabled `calculateField` and `merge` transformations after setting expressions.

- **D-06:** **Panels 1, 2, 3, 7, 9, 38-44 (merge+reduce sum panels):** Add an Expression target with `expr = $A + $B`. Remove `merge` and `reduce` transformations. The panel then reads the Expression target's output as its single display value.

### Approach for Broken Panels (5, 6, 10)

- **D-07:** Fix by filling in the blank `expr` fields on existing Expression targets. These panels were partially migrated post-v1 but left with empty formulas — complete the migration rather than rebuilding from scratch.

- **D-08:** Remove the disabled `calculateField` and `merge` transforms from panel 6 after setting correct expressions — clean up leftover disabled transforms to reduce JSON noise.

### Agent's Discretion
- Exact Expression target JSON structure (`datasource`, `hide`, `refId`, `type: "math"`)
- Whether to set `hide: true` on the raw InfluxDB queries (A/B/East/West/Grid) after adding expressions — common pattern to hide intermediate queries from panel display
- Order of targets array in the panel JSON (convention: raw queries first, then expression targets)
- Whether panels that already display correctly (reduce sum panels) need the `reduceOptions` updated after switching to Expression

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project Specs
- `.planning/PROJECT.md` — InfluxDB schema (measurement names, field names), datasource name `influxdb-solar`
- `.planning/REQUIREMENTS.md` — Requirements context for affected panels (OVER-05, OVER-06)

### Existing Dashboard
- `solar-pv-monitor.json` — The dashboard to be modified. All panel edits happen in this file.

### Prior Phase Context
- `.planning/phases/04-financial-savings-canvas-layout-polish/04-CONTEXT.md` — D-57 (production color scale), D-55 (TOU timezone handling) — relevant if any financial panel queries need checking
- `.planning/phases/01-foundation-overview-stats/01-CONTEXT.md` — D-08 (`__inputs` pattern), D-13 (COALESCE), D-14 (color scheme) — original panel design decisions

### Post-v1 Dashboard Changes (in ROADMAP.md)
- `.planning/ROADMAP.md` §"Post-v1 Dashboard Changes" — Documents the partial House Load / Self-Consumption expression migration that this phase completes

No external API or Grafana docs reference needed — Expression target (type=math) pattern is already implemented in the current dashboard (panels 5/6/10 have the targets, just with empty expr).

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- Panels 5 and 10 already have Expression targets with `type: "math"` — use these as the template for Expression target JSON structure
- Panel 6 already has 3 Expression targets (Solar, Total, Self) — just need the `expr` fields filled in
- Pattern: raw InfluxDB query targets (refId A/B or East/West/Grid) → Expression target reads those via `$refId`

### Established Patterns
- Expression targets in Grafana use `"datasource": {"type": "__expr__", "uid": "__expr__"}` and `"type": "math"` with `"expr": "$A + $B"`
- After expression migration: raw query targets typically have `"hide": true` to suppress them from panel display
- The `merge` transform is removed once an Expression target produces the final calculated value
- The `reduce` transform (for sum) is replaced by the Expression `$A + $B` pattern

### Integration Points
- All panels are in `solar-pv-monitor.json` — single file edit
- Panel IDs affected: 1, 2, 3, 5, 6, 7, 9, 10, 38, 39, 40, 41, 42, 43, 44 (15 panels)
- No new panels — this is a within-panel migration only
- No row structure changes needed

</code_context>

<specifics>
## Specific Ideas

- User's intent: "reduce usage of transformation if it's using for calculation and use math expression instead (calculate on database)" — the phrase "calculate on database" is slightly imprecise; Grafana Expressions run server-side in Grafana, not in InfluxDB. But the intent is clear: move calculation logic out of client-side transformation pipeline.
- Panels 5, 6, 10 are the primary examples the user cited. The blank `expr` fields on those panels confirm they were partially migrated and left incomplete.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 05-fix-all-panel-in-dashboard-that-still-use-transformation-to-calcuate-to-use-expression-instead*
*Context gathered: 2026-04-01*
