---
phase: 09-update-panels-to-use-new-ct-meter-data-for-grid-energy-import-current-voltage-real-power-instead-of-smart-meter
verified: 2026-04-04T00:00:00Z
status: passed
score: 11/11 must-haves verified
re_verification: false
---

# Phase 9: CT Grid Meter Migration Verification Report

**Phase Goal:** Migrate 12 dashboard panels from Smart Meter to CT grid meter (`grid` table) for grid power, voltage, and current data — leaving frequency and power_factor panels on Smart Meter since those fields don't exist in the CT table.
**Verified:** 2026-04-04
**Status:** ✅ passed
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Grid Import stat (Panel 4) reads from the CT grid table, not the Smart Meter | ✓ VERIFIED | `SELECT power AS grid_kw FROM "grid" WHERE time >= now() - INTERVAL '2 minutes' ORDER BY time DESC LIMIT 1` — exact plan SQL confirmed |
| 2 | House ← Grid power flow stat (Panel 11) reads from the CT grid table | ✓ VERIFIED | Identical SQL to Panel 4: `SELECT power AS grid_kw FROM "grid" ...` |
| 3 | Grid Voltage stat (Panel 16) reads from the CT grid table | ✓ VERIFIED | `SELECT voltage AS voltage FROM "grid" WHERE time >= now() - INTERVAL '2 minutes' ORDER BY time DESC LIMIT 1` |
| 4 | Grid Current stat (Panel 19) reads from the CT grid table | ✓ VERIFIED | `SELECT current AS current FROM "grid" WHERE time >= now() - INTERVAL '2 minutes' ORDER BY time DESC LIMIT 1` |
| 5 | Voltage & Frequency chart (Panel 20) uses CT voltage; Frequency series stays on Smart Meter | ✓ VERIFIED | refId=A: `AVG(voltage) FROM "grid"` (CT); refId=B: `AVG(frequency_sensor) FROM "Smart Meter"` (preserved) |
| 6 | Power Factor & Current chart (Panel 21) uses CT current; Power Factor series stays on Smart Meter | ✓ VERIFIED | refId=B: `AVG(current) FROM "grid"` (CT); refId=A: `AVG(power_factor_sensor) FROM "Smart Meter"` (preserved) |
| 7 | Self-Consumption stat (Panel 6) Grid query component reads from CT grid table | ✓ VERIFIED | refId=Grid: `COALESCE(selector_last(power, time)['value'], 0) AS grid_w FROM "grid"` |
| 8 | Calculated Load reconciliation stat (Panel 905) Grid query component reads from CT grid table | ✓ VERIFIED | refId=Grid: `COALESCE(selector_last(power, time)['value'], 0) AS grid_w FROM "grid"` |
| 9 | Power Profile chart (Panel 801) Grid series reads from CT grid table without GREATEST() clip | ✓ VERIFIED | `AVG(power) / 1000.0 AS "Grid" FROM "grid" ...` — no GREATEST(); negative kW now visible during backfeed |
| 10 | Backfeed panels (33/34/35) detect and log backfeed using CT grid.power < 0 instead of Smart Meter | ✓ VERIFIED | All three panels: `FROM "grid" WHERE ... AND power < 0` |
| 11 | Panels 17 (Frequency) and 18 (Power Factor) remain unchanged on Smart Meter | ✓ VERIFIED | Panel 17: `frequency_sensor FROM "Smart Meter"`; Panel 18: `power_factor_sensor FROM "Smart Meter"` — neither touched |

**Score:** 11/11 truths verified

---

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `solar-pv-monitor.json` | Updated dashboard with 12 panels switched from Smart Meter to CT grid table | ✓ VERIFIED | JSON is valid, dashboard version bumped 45→46, `"grid"` table referenced in 12 targets, `"Smart Meter"` remains in 5 rawSql targets (intentional) |

**Artifact level checks:**
- **Level 1 (Exists):** ✓ File present
- **Level 2 (Substantive):** ✓ Contains `FROM "grid"` in 12 query targets; Smart Meter count reduced from 18→5 in rawSql
- **Level 3 (Wired):** ✓ Dashboard importable JSON (`schemaVersion: 42`, `version: 46`); commit `f5836da` confirmed in git log

---

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| Panel 4 (Grid Import), Panel 11 (House ← Grid) | grid table power field | `SELECT power AS grid_kw FROM "grid" WHERE time >= now() - INTERVAL '2 minutes' ORDER BY time DESC LIMIT 1` | ✓ WIRED | Exact SQL from plan confirmed in both panels |
| Panel 6 (Self-Consumption), Panel 905 (Calculated Load) | grid table power field via Expression | `COALESCE(selector_last(power, time)['value'], 0) AS grid_w FROM "grid"` | ✓ WIRED | Both panels' refId=Grid target confirmed |
| Panel 801 Grid series | grid table power field (signed, kW scale) | `AVG(power) / 1000.0 AS "Grid" FROM "grid"` — no GREATEST() clip | ✓ WIRED | GREATEST removed; negative values enabled for backfeed visualization |
| Panels 33/34/35 (Backfeed) | grid table power field | `power < 0 FROM "grid"` (signed CT measurement) | ✓ WIRED | All three panels use `"grid"` table with `power < 0` condition |

---

### Data-Flow Trace (Level 4)

> Step 7b / Level 4 behavioral checks: This phase modifies only `rawSql` strings in a static JSON dashboard file — there is no running application to probe. All data-flow correctness is verified by inspecting the SQL query strings against the known CT grid table schema (`power`, `voltage`, `current` fields, no `_sensor` suffix).

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|--------------------|--------|
| Panel 4 / Panel 11 | `grid_kw` | `"grid"` table `power` field | ✓ Field exists in CT schema (W, signed) | ✓ FLOWING |
| Panel 16 | `voltage` | `"grid"` table `voltage` field | ✓ Field exists in CT schema (V, always positive) | ✓ FLOWING |
| Panel 19 | `current` | `"grid"` table `current` field | ✓ Field exists in CT schema (A, signed) | ✓ FLOWING |
| Panel 20 refId=A | `Voltage` time-series | `"grid"` table `voltage` via DATE_BIN | ✓ Correct field, correct aggregation | ✓ FLOWING |
| Panel 21 refId=B | `Current` time-series | `"grid"` table `current` via DATE_BIN | ✓ Correct field, correct aggregation | ✓ FLOWING |
| Panel 801 Grid series | `Grid` kW time-series | `"grid"` table `power` / 1000.0 | ✓ Signed power enables backfeed negative | ✓ FLOWING |
| Panels 33/34/35 | Count/Max/Log | `"grid"` table `power` with `power < 0` | ✓ Signed CT power correctly filters backfeed events | ✓ FLOWING |

---

### Behavioral Spot-Checks

> Step 7b: SKIPPED (static JSON dashboard file — no runnable server entry points to probe directly)

---

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| GRID-01 | 09-01-PLAN.md | User can see grid voltage (V) and frequency (Hz) as stats | ✓ SATISFIED | Panel 16 (voltage from CT `"grid"`), Panel 17 (frequency stays on Smart Meter). Both stat panels present with correct data sources. |
| GRID-02 | 09-01-PLAN.md | Smart meter detailed view: power factor, current, voltage, frequency time-series | ✓ SATISFIED | Panel 20: Voltage refId=A (CT `"grid"`), Frequency refId=B (Smart Meter preserved). Panel 21: Current refId=B (CT `"grid"`), Power Factor refId=A (Smart Meter preserved). Mixed-source pattern correctly implemented. |
| OVER-04 | 09-01-PLAN.md | Current grid import power (kW) as a stat | ✓ SATISFIED | Panel 4 confirmed reading `power AS grid_kw FROM "grid"` — now sourced from more accurate CT sensor. |
| OVER-05 | 09-01-PLAN.md | Current house load (kW) = solar production + grid import | ✓ SATISFIED | Panel 905 refId=Grid now reads from CT `"grid"` table; math Expression `$East + $West + $Grid` unchanged. Load calculation remains correct with more accurate grid data source. |
| OVER-06 | 09-01-PLAN.md | Self-consumption ratio (%) showing how much load is covered by solar | ✓ SATISFIED | Panel 6 refId=Grid now reads from CT `"grid"` table; Expression chain `$Solar / $Total` unchanged. Ratio calculation remains correct. |
| EVNT-01 | 09-01-PLAN.md | Grid backfeed event log: timestamp, power, duration for Smart Meter power_sensor < 0 | ✓ SATISFIED | Panel 35 now queries `FROM "grid" WHERE ... AND power < 0` — CT signed power enables more accurate backfeed detection. Log shows ABS(power), voltage, current from CT. |
| EVNT-02 | 09-01-PLAN.md | Today's backfeed event count and max backfeed power as summary stats | ✓ SATISFIED | Panel 33 (COUNT from "grid" WHERE power < 0) and Panel 34 (ABS(MIN(power)) from "grid" WHERE power < 0) both confirmed. |

**Requirement traceability note:** REQUIREMENTS.md maps GRID-01/GRID-02 to Phase 2, OVER-04/05/06 to Phase 1, and EVNT-01/EVNT-02 to Phase 3 as their originating phases. Phase 9 is a targeted data-source migration that re-satisfies these same requirements using more accurate CT sensor data. The traceability table in REQUIREMENTS.md correctly reflects the originating phases; Phase 9 represents an enhancement that does not require updating that table.

**Orphaned requirements check:** No requirements in REQUIREMENTS.md are mapped exclusively to Phase 9 — all 7 requirement IDs claimed in the plan belong to earlier originating phases and are re-satisfied here. No orphaned Phase 9 requirements detected.

---

### Anti-Patterns Found

| File | Pattern | Severity | Impact |
|------|---------|----------|--------|
| `solar-pv-monitor.json` Panel 33 description | Description text still says "Smart Meter power < 0" but panel now queries CT `"grid"` table | ℹ Info | Cosmetic only — the query is correct; description is stale metadata |

**Verdict:** No blockers, no warnings. One info-level cosmetic issue (stale description text).

**Smart Meter reference audit:**
- 5 rawSql references: Intentional — Panels 17, 18, 20 refId=B, 21 refId=A, 801 refId=Load
- 1 description text reference: Panel 33 description mentions "Smart Meter power < 0" (stale, not a query)
- 0 unintended rawSql references: All migrated queries confirmed on CT `"grid"` table
- No `_sensor` suffixes in CT grid table queries

---

### Human Verification Required

#### 1. Live Panel Data Validation

**Test:** Import `solar-pv-monitor.json` (v46) into Grafana and inspect the 12 migrated panels during daylight/grid-active hours.
**Expected:**
- Panels 4 and 11: Grid import stat shows a positive watt value consistent with prior Smart Meter readings (±1-2%)
- Panel 16: Grid voltage ~237V from CT sensor
- Panel 19: Grid current shows positive ampere value
- Panel 20: Voltage series plots CT data; Frequency (50 Hz) series plots Smart Meter data (both visible)
- Panel 21: Current series plots CT data; Power Factor series plots Smart Meter data (both visible)
- Panel 6 / Panel 905: Self-consumption % and Calculated Load remain plausible values (no NaN/Inf)
- Panel 801: Grid line visible; during backfeed events, Grid line dips below 0 kW (new behavior, correct)
- Panels 33/34/35: Backfeed log reflects CT measurements
- Panels 17/18: Frequency (~50 Hz) and Power Factor still show from Smart Meter
**Why human:** Runtime query execution, live CT vs Smart Meter value comparison, and visual chart integrity cannot be verified from static JSON inspection alone.

---

### Gaps Summary

No gaps found. All 11 must-have truths are verified against the actual `solar-pv-monitor.json` content. All 4 key links are confirmed wired. All 7 requirements are satisfied. The single anti-pattern found (Panel 33 stale description text) is cosmetic and does not affect query correctness or goal achievement.

---

## Summary

Phase 9 successfully migrated 12 dashboard panels from Smart Meter to the CT grid meter (`"grid"` table):

- **10 surgical SQL replacements** applied across Panels 4, 6, 11, 16, 19, 20 (refId=A), 21 (refId=B), 33, 34, 35, 801 (refId=Grid), 905 (refId=Grid)
- **Mixed-source panels preserved correctly:** Panel 20 refId=B (Frequency) and Panel 21 refId=A (Power Factor) correctly retained on Smart Meter
- **Panels 17 & 18 untouched:** Frequency and Power Factor stat panels remain on Smart Meter (CT table has no these fields)
- **Panel 801 Grid series:** GREATEST() clip removed — signed negative kW now visible during backfeed periods
- **Dashboard version bumped:** 45 → 46
- **JSON validity:** Confirmed parseable and schema-valid
- **Commit verified:** `f5836da` present in git log

The phase goal is fully achieved.

---

_Verified: 2026-04-04_
_Verifier: the agent (gsd-verifier)_
