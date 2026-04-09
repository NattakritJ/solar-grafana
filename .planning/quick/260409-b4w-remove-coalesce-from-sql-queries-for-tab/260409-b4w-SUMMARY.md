---
id: 260409-b4w
type: quick
date: "2026-04-09"
duration: "2min"
files_modified: 1
commits: 1
tags: [sql, cleanup, coalesce, influxdb, dashboard]
key_files:
  modified:
    - solar-pv-monitor.json
decisions:
  - "Used 3 targeted sed passes (not 2) because the grid table name is JSON-escaped as \\\"grid\\\" in the file, requiring a separate pass to match the escaped form"
---

# Quick Task 260409-b4w: Remove COALESCE from SQL Queries for Target Tables Summary

**One-liner:** Removed 21 `COALESCE(AVG/MAX(power), 0)` wrappers from 6 always-sending InfluxDB tables, leaving 44 COALESCE usages in other panels untouched.

## What Was Done

Stripped unnecessary `COALESCE(..., 0)` fallback wrappers from every `rawSql` query that targets these 6 tables which always produce data (never null):

| Table | Occurrences Removed |
|-------|-------------------|
| `east_microinverter_power` | 5 |
| `west_microinverter_power` | 5 |
| `"grid"` | 2 |
| `235_floor_1` | 3 |
| `235_floor_2` | 3 |
| `236_floor_1` | 3 |
| **Total** | **21** |

### Pattern Applied

```sql
-- Before:
SELECT COALESCE(AVG(power), 0) AS east_w FROM east_microinverter_power ...
SELECT COALESCE(MAX(power), 0) AS peak_kw FROM east_microinverter_power ...

-- After:
SELECT AVG(power) AS east_w FROM east_microinverter_power ...
SELECT MAX(power) AS peak_kw FROM east_microinverter_power ...
```

All SQL aliases (AS east_w, AS grid_w, AS floor1_w, etc.) were preserved exactly.

## Verification Results

| Check | Expected | Actual | Status |
|-------|----------|--------|--------|
| COALESCE in target tables | 0 | 0 | ✅ |
| Total COALESCE remaining | 44 | 44 | ✅ |
| JSON valid | true | true | ✅ |

## Commits

| Hash | Description |
|------|-------------|
| fa67d80 | fix(260409-b4w): remove COALESCE from 6 always-sending table queries |

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Extra sed pass needed for JSON-escaped grid table name**
- **Found during:** Task 1 — after the 2 planned sed passes, 46 COALESCE remained instead of 44
- **Issue:** The grid table `"grid"` is stored in JSON as `\"grid\"` (escaped double quotes). The planned sed pattern `/east_microinverter_power|...|"grid"|.../` matched literal `"grid"` but the file contains `\"grid\"`, so the 2 grid-table COALESCE occurrences were missed
- **Fix:** Added a 3rd sed pass targeting `FROM \\"grid\\"` to catch the escaped form
- **Files modified:** solar-pv-monitor.json
- **Commit:** fa67d80

## Self-Check: PASSED

- [x] solar-pv-monitor.json modified ✅
- [x] Commit fa67d80 exists ✅
- [x] 0 COALESCE in target tables ✅
- [x] 44 COALESCE remaining in other tables ✅
- [x] JSON valid ✅
