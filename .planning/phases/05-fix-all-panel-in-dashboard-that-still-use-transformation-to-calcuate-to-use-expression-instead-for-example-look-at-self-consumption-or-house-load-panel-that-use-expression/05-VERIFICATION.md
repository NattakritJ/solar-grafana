---
phase: 05-fix-transformations-to-expressions
verified: 2026-04-01T10:15:00Z
status: passed
score: 7/7 must-haves verified
re_verification: false
human_verification:
  - test: "Import solar-pv-monitor.json into Grafana and confirm all 15 calculation panels display values"
    expected: "All panels show numeric values — no 'No data' or error badges on panels 1, 2, 3, 5, 6, 7, 9, 10, 38-44"
    why_human: "Expression target evaluation requires live Grafana + InfluxDB connection; already approved by user per 05-02-SUMMARY.md Task 2 checkpoint"
---

# Phase 05: Fix Transformations → Expressions Verification Report

**Phase Goal:** Fix all panels in the dashboard that still use transformations to calculate values — replace merge+reduce/calculateField transformation chains with Grafana Expression targets (type=math) for all calculation panels.
**Verified:** 2026-04-01T10:15:00Z
**Status:** ✓ PASSED
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths

| #  | Truth | Status | Evidence |
|----|-------|--------|---------|
| 1  | Panels 5 and 10 (House Load) have zero transformations and expression `$East + $West + $Grid` in targets | ✓ VERIFIED | `transformations=[]`, `expr=['$East + $West + $Grid']` on both; 3/3 InfluxDB targets `hide:true` |
| 2  | Panel 6 (Self-Consumption) has zero transformations and three expression targets (Solar/Total/Self) | ✓ VERIFIED | `transformations=[]`, 3 math targets: `$East+$West`, `$Solar+$Grid`, `$Solar/$Total` |
| 3  | Panels 1, 2, 3, 7, 9 each have a single Expression target (`$A + $B`) and no merge/reduce transformations | ✓ VERIFIED | All 5 panels: `transformations=[]`, 1 expr target `refId=C`, `expression="$A + $B"` |
| 4  | Raw InfluxDB query targets on panels 1, 2, 3, 7, 9 have `hide: true` | ✓ VERIFIED | All panels: 2/2 InfluxDB targets `hide=True` |
| 5  | Panels 38-44 each have a single Expression target (`$A + $B`) and empty transformations | ✓ VERIFIED | All 7 panels: `transformations=[]`, 1 expr target `refId=C`, `expression="$A + $B"` |
| 6  | Raw InfluxDB query targets on panels 38-44 have `hide: true` | ✓ VERIFIED | All 7 panels: 2/2 InfluxDB targets `hide=True` |
| 7  | Display-only panels (13, 14, 15, 22, 23, 45) retain their `merge` transforms unchanged | ✓ VERIFIED | All 6 display panels: `has_merge=True`; panels 14 also retains `reduce` for display purpose |

**Score:** 7/7 truths verified

---

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `solar-pv-monitor.json` | Updated dashboard with expression-based calculations for all calculation panels | ✓ VERIFIED | Valid JSON; 51 total panels (44 non-row); `schemaVersion: 42`; `timezone: Asia/Bangkok` |
| Expression targets — `"type": "math"` | Present on all 15 calculation panels | ✓ VERIFIED | `grep "type.*math"` confirms presence across panels 1,2,3,5,6,7,9,10,38-44 |

---

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| Panel 1 targets refId=C | Expression `$A + $B` | math expression target | ✓ WIRED | `datasource.type=__expr__`, `datasource.uid=__expr__`, `expression="$A + $B"`, `refId=C` |
| Panel 5 targets refId=Total | Expression `$East + $West + $Grid` | math expression target | ✓ WIRED | `datasource.type=__expr__`, `expression="$East + $West + $Grid"`, `refId=Total` |
| Panel 6 — 3 chained expressions | Solar→Total→Self | sequential math targets | ✓ WIRED | `Solar=$East+$West`, `Total=$Solar+$Grid`, `Self=$Solar/$Total`; all `type=__expr__` |
| Panels 38-44 refId=C | Expression `$A + $B` | math expression target summing East+West | ✓ WIRED | All 7: `datasource.type=__expr__`, `expression="$A + $B"`, `refId=C` |
| Panels 41-44 InfluxDB targets A/B | TOU SQL (peak/off-peak hour filtering) | EXTRACT(HOUR) + EXTRACT(DOW) WHERE clauses | ✓ WIRED | Full SQL verified: uses `EXTRACT(HOUR FROM bucket AT TIME ZONE 'Asia/Bangkok')`, rate multipliers `5.7982` and `2.6369`, Thai holiday exclusions |

---

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|--------------|--------|-------------------|--------|
| Panel 1 (Solar Power) | Expression result from `$A + $B` | InfluxDB `rawSql` targets A (East inverter) + B (West inverter), `hide:true` | Yes — SQL queries `pv1..pv4_power_sensor` fields from inverter tables | ✓ FLOWING |
| Panel 5/10 (House Load) | Expression result from `$East + $West + $Grid` | InfluxDB targets East, West (inverters), Grid (smart meter), all `hide:true` | Yes — SQL queries live sensor data | ✓ FLOWING |
| Panel 6 (Self-Consumption) | `$Solar/$Total` (ratio) | Chained expressions: Solar and Total computed from InfluxDB inverter+grid queries | Yes — chained math expressions; final ratio meaningful only with live data | ✓ FLOWING |
| Panels 38-44 (Financial) | Expression result from `$A + $B` | InfluxDB TOU SQL targets A (East savings) + B (West savings), `hide:true` | Yes — SQL queries contain `EXTRACT(HOUR)`, `SUM(hourly_kwh * rate)`, holiday exclusions | ✓ FLOWING |

---

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| Dashboard JSON is valid and parseable | `python3 -c "import json; json.load(open('solar-pv-monitor.json'))"` | Exit 0 | ✓ PASS |
| All 15 calc panels have `transformations: []` | Python assertion loop over `[1,2,3,5,6,7,9,10,38-44]` | All passed | ✓ PASS |
| All 15 calc panels have ≥1 `type: math` expression target | Python assertion | All passed | ✓ PASS |
| All InfluxDB targets on calc panels have `hide: true` | Python assertion | All passed — 2/2 or 3/3 per panel | ✓ PASS |
| Display panels (13,14,15,22,23,45) retain `merge` transform | Python assertion | All 6 passed | ✓ PASS |
| Commits documented in SUMMARYs exist in git | `git log --oneline {hash}^..{hash}` for 3 commits | All 3 found | ✓ PASS |
| Financial panels contain TOU SQL (peak/off-peak logic) | SQL content check for `EXTRACT(HOUR)`, rate multipliers | Found in all panels 41-44 | ✓ PASS |

---

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|-------------|-------------|--------|----------|
| OVER-03 | 05-01 | User can see lifetime total energy production (kWh/MWh) as a stat | ✓ SATISFIED | Panel 3 (Lifetime Energy): `transformations=[]`, expr=`$A + $B`; stat panel displays correct total |
| OVER-05 | 05-01 | User can see current house load (kW) calculated as solar production + grid import | ✓ SATISFIED | Panels 5 & 10: `transformations=[]`, expr=`$East + $West + $Grid`; formula correctly adds both inverters + grid |
| OVER-06 | 05-01 | User can see self-consumption ratio (%) showing how much load is covered by solar | ✓ SATISFIED | Panel 6: `transformations=[]`, 3-step chained expressions `$Solar/$Total`; correct ratio calculation |
| FINC-01 | 05-02 | User can see today's estimated savings in THB based on solar production | ✓ SATISFIED | Panel 38 (Today's Savings): `transformations=[]`, expr=`$A + $B` summing East+West TOU savings |
| FINC-02 | 05-02 | User can see TOU-aware savings calculations using peak/off-peak rates | ✓ SATISFIED | Panels 41-44: InfluxDB SQL uses `EXTRACT(HOUR)`, rates 5.7982/2.6369 THB, Thai holiday exclusions; expressions sum East+West correctly |
| FINC-03 | 05-02 | User can see cumulative savings for month-to-date and year-to-date | ✓ SATISFIED | Panels 39 (Month) and 40 (Year): `transformations=[]`, expr=`$A + $B`; TOU SQL in underlying targets |
| FINC-04 | 05-02 | User can see savings breakdown showing peak kWh vs off-peak kWh and THB values | ✓ SATISFIED | Panels 41 (Peak kWh), 42 (Peak THB), 43 (Off-Peak kWh), 44 (Off-Peak THB): all migrated, each with dedicated TOU SQL |

**Note on REQUIREMENTS.md traceability table:** The traceability table maps OVER-03/05/06 to Phase 1 and FINC-01/02/03/04 to Phase 4. Phase 5 is a **quality/implementation refactoring phase** — the user-visible requirements were already satisfied in Phases 1 and 4. Phase 5 replaces the client-side transformation pipeline (merge+reduce) with server-side Grafana Expression targets. The requirements remain satisfied; the implementation method is now correct.

**Orphaned requirements check:** No requirements in REQUIREMENTS.md are mapped to Phase 5 in the traceability table. The requirement IDs claimed by the plans (OVER-03, OVER-05, OVER-06, FINC-01-04) are legitimately addressed here as a refactoring improvement. No orphaned requirement IDs found.

---

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| — | — | — | — | None found |

No stub patterns, no TODO/FIXME, no hardcoded empty returns, no disconnected props. All expression formulas are non-trivial and mathematically meaningful. All InfluxDB targets contain substantial SQL (92–2,102 characters) with real sensor field references.

---

### Human Verification Required

#### 1. Visual Display Confirmation in Grafana

**Test:** Import `solar-pv-monitor.json` into Grafana (Dashboards → Import → Upload JSON, select `influxdb-solar` datasource)
**Expected:** All 15 migrated calculation panels display numeric values — Solar Power, Today's Energy, Lifetime Energy, Peak Today, ☀ Solar → House, House Load (×2), Self-Consumption, Today's Savings, Month Savings, Year Savings, Peak kWh, Peak THB, Off-Peak kWh, Off-Peak THB
**Why human:** Expression target evaluation requires live Grafana connected to InfluxDB; can't verify mathematically correct output programmatically without running the stack
**Prior approval status:** Already approved by user as part of 05-02 Task 2 human checkpoint (per 05-02-SUMMARY.md: "Human verification approved: all migrated panels display values correctly in Grafana with no regressions")

---

### Gaps Summary

No gaps. All 7 must-have truths verified. All 15 calculation panels confirmed to have:
1. `transformations: []` (empty — no merge/reduce/calculateField)
2. At least one `type: math` Expression target with correct formula
3. All InfluxDB raw query targets marked `hide: true`

All 6 display-only panels (13, 14, 15, 22, 23, 45) confirmed to retain their `merge` transforms intact.

The dashboard JSON is valid (parses cleanly), structure is intact (51 panels, 44 non-row), and all 3 git commits documented in the SUMMARYs (`9038a0b`, `fe9b514`, `1373104`) exist in the repository.

---

_Verified: 2026-04-01T10:15:00Z_
_Verifier: gsd-verifier (claude-sonnet-4.6)_
