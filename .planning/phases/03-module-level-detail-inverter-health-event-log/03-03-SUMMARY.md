---
phase: 03-module-level-detail-inverter-health-event-log
plan: 03
subsystem: dashboard
tags: [grafana, table, stat, event-log, backfeed, alarm, fault, unified-log]

requires:
  - phase: 03-module-level-detail-inverter-health-event-log
    provides: Inverter health panels (24-32) in uncollapsed Row 500
provides:
  - Today's backfeed count and max backfeed power summary stats
  - Grid backfeed event log table with timestamps and power values
  - Alarm/fault history table for both inverters
  - Unified event log combining backfeed, alarm, fault, and state changes chronologically
affects: [phase-4-polish]

tech-stack:
  added: []
  patterns: [negative power detection (power_sensor < 0), ABS for display, multi-source merge for unified log, CASE WHEN for event type classification]

key-files:
  created: []
  modified: [solar-pv-monitor.json]

key-decisions:
  - "Backfeed defined as power_sensor < 0 from Smart Meter (D-38)"
  - "Backfeed power displayed as ABS value to avoid negative number confusion (D-41)"
  - "Unified event log uses 3 separate queries + merge transformation, no UNION ALL (D-43)"
  - "Non-producing state (device_state_sensor != 2) logged as 'State Change' in unified log"
  - "Color coding: backfeed=purple, alarm=orange, fault=red, state change=blue"
  - "Source colors: Smart Meter=purple, East=orange (#FF9830), West=blue (#5794F2)"

patterns-established:
  - "Backfeed detection: WHERE power_sensor < 0 on Smart Meter measurement"
  - "Multi-source unified log: separate queries per event source, merge + sortBy transformations"
  - "CASE WHEN for event type classification in SQL"
  - "ABS() for displaying negative values as positive"

requirements-completed: [HLTH-03, EVNT-01, EVNT-02, EVNT-03]

duration: 3min
completed: 2026-03-30
---

# Phase 03 Plan 03: Backfeed Stats, Event Logs, and Unified Event Log Summary

**Grid backfeed detection with summary stats and event logs, plus alarm/fault history and unified chronological event log from all sources**

## Performance

- **Duration:** 3 min
- **Started:** 2026-03-30T05:48:00Z
- **Completed:** 2026-03-30T05:51:00Z
- **Tasks:** 2/2
- **Files modified:** 1

## Accomplishments

### Task 1: Uncollapse Event Log row and add backfeed summary stats + backfeed event log
- Set Row 700 (Event Log) to `collapsed: false`
- Added Panel 33 (stat): "Today's Backfeed Count" — COUNT(*) WHERE power_sensor < 0
- Added Panel 34 (stat): "Max Backfeed Power" — ABS(MIN(power_sensor)) WHERE power_sensor < 0
- Added Panel 35 (table): "Grid Backfeed Log" — shows Time, Export Power (W), Voltage (V), Current (A)
- Export Power uses ABS(power_sensor) for positive display
- Color thresholds on export power: green < 100W, yellow 100-500W, red > 500W

### Task 2: Add alarm/fault history log and unified event log
- Added Panel 36 (table): "Alarm / Fault History" — 2 queries (East/West) filtering alarm_sensor != 0 OR fault_sensor != 0
- CASE WHEN classifies events as Alarm, Fault, Alarm + Fault, or State Change
- Alarm Code and Fault Code columns color-coded (green=0, red=non-zero)
- Added Panel 37 (table): "Event Log" — unified log with 3 queries (Backfeed, East events, West events)
- Merge + sortBy transformations combine all sources chronologically
- Source column color-coded: Smart Meter=purple, East=orange, West=blue
- Event column color-coded: Backfeed=purple, Alarm=orange, Fault=red, State Change=blue

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| 1+2 | 19ab3a7 | feat(03-03): add backfeed stats, event logs, alarm/fault history, and unified event log |

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — all panels are wired to live InfluxDB queries.

## Self-Check: PASSED
