---
phase: "05"
plan: "01"
subsystem: dashboard
tags: [expressions, transformations, migration, panels]
dependency_graph:
  requires: []
  provides: [expression-based-stat-panels]
  affects: [solar-pv-monitor.json]
tech_stack:
  added: []
  patterns: [grafana-expression-target, type-math, hide-raw-queries]
key_files:
  created: []
  modified:
    - solar-pv-monitor.json
decisions:
  - "Remove transformation chains (merge+reduce/calculateField) from 8 stat panels and replace with Grafana Expression targets (type=math)"
  - "InfluxDB query targets on stat panels set to hide:true once Expression target provides the calculated value"
  - "Display-only merge panels (13, 14, 15, 22, 23, 45) untouched — merge for display is acceptable"
metrics:
  duration: "2 minutes"
  completed_date: "2026-04-01"
  tasks_completed: 2
  files_modified: 1
---

# Phase 05 Plan 01: Fix Transformation-Based Calculations → Expressions Summary

**One-liner:** Migrated 8 overview/power-flow stat panels from client-side merge+reduce/calculateField transformation chains to server-side Grafana Expression targets (type=math, `$A + $B` formula).

## What Was Built

Cleaned up leftover transformations from panels 5, 6, 10 (already partially migrated post-v1) and fully migrated panels 1, 2, 3, 7, 9 from `merge + reduce(sum)` transformation pipelines to explicit Grafana Expression targets.

### Changes Made

| Panel | Title | Before | After |
|-------|-------|--------|-------|
| 5 | House Load | `transformations: [merge]` | `transformations: []` |
| 6 | Self-Consumption | `transformations: [merge(disabled), calculateField×3(disabled), filterFieldsByName(disabled)]` | `transformations: []` |
| 10 | 🏠 House Load | `transformations: [merge]` | `transformations: []` |
| 1 | Solar Power | `transformations: [merge, reduce(sum)]` | `transformations: []` + Expression `$A + $B` |
| 2 | Today's Energy | `transformations: [merge, reduce(sum)]` | `transformations: []` + Expression `$A + $B` |
| 3 | Lifetime Energy | `transformations: [merge, reduce(sum)]` | `transformations: []` + Expression `$A + $B` |
| 7 | Peak Today | `transformations: [merge, reduce(sum)]` | `transformations: []` + Expression `$A + $B` |
| 9 | ☀ Solar → House | `transformations: [merge, reduce(sum)]` | `transformations: []` + Expression `$A + $B` |

**Out-of-scope panels preserved:** Panels 13, 14, 15, 22, 23, 45 retain their display-only `merge` transformations (unchanged).

### Expression Target Pattern Applied

```json
{
  "datasource": {
    "name": "Expression",
    "type": "__expr__",
    "uid": "__expr__"
  },
  "expression": "$A + $B",
  "refId": "C",
  "type": "math"
}
```

Raw InfluxDB query targets (refId=A, refId=B) have `"hide": true` to suppress them from panel display.

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| Task 1 | `9038a0b` | Clean up leftover transformations from panels 5, 6, 10 |
| Task 2 | `fe9b514` | Migrate panels 1, 2, 3, 7, 9 from merge+reduce to Expression targets |

## Deviations from Plan

None — plan executed exactly as written.

Panel 6 already had correct expression values (`$East + $West`, `$Solar + $Grid`, `$Solar / $Total`) filled in from the post-v1 edits. Task 1 only needed to clear the leftover disabled transformations, as specified.

## Known Stubs

None — all Expression targets have correct formulas wired to live InfluxDB query targets.

## Self-Check: PASSED

| Item | Status |
|------|--------|
| `solar-pv-monitor.json` | FOUND |
| `05-01-SUMMARY.md` | FOUND |
| Commit `9038a0b` (Task 1) | FOUND |
| Commit `fe9b514` (Task 2) | FOUND |
