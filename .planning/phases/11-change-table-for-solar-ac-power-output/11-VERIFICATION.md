---
phase: 11-change-table-for-solar-ac-power-output
verified: 2026-04-09T00:30:00Z
status: human_needed
score: 11/11 must-haves verified (automated)
human_verification:
  - test: "Import dashboard and verify CT solar panels show live data"
    expected: "Panels 1, 6, 7, 9, 12, 13, 801, 905 display real-time CT power readings; pre-existing CT panels update at 5s refresh; unchanged panels still work"
    why_human: "Cannot verify live InfluxDB data flow or visual correctness programmatically"
---

# Phase 11: Change Table for Solar AC Power Output — Verification Report

**Phase Goal:** Migrate 8 panels from slow-polling inverter tables (`"East Microinverter"` / `"West Microinverter"`) to fast CT sensor tables (`east_microinverter_power` / `west_microinverter_power`), tighten recency windows on all CT snapshot panels from 1 minute to 10 seconds, and increase dashboard refresh from 10s to 5s

**Verified:** 2026-04-09T00:30:00Z
**Status:** human_needed
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Solar AC Power stat (Panel 1) reads from east/west_microinverter_power CT tables instead of East/West Microinverter | ✓ VERIFIED | Panel 1 refId=A: `AVG(power) ... FROM east_microinverter_power`, refId=B: `FROM west_microinverter_power`. No `power_sensor` or `"East Microinverter"` in these targets. |
| 2 | Self-Consumption stat (Panel 6) East/West targets read from CT tables | ✓ VERIFIED | Panel 6 refId=East: `east_w FROM east_microinverter_power`, refId=West: `west_w FROM west_microinverter_power`. Grid target also at 10s window. |
| 3 | Peak AC Power Today stat (Panel 7) reads from CT tables | ✓ VERIFIED | Panel 7 refId=A: `MAX(power) ... FROM east_microinverter_power`, refId=B: `FROM west_microinverter_power`. Uses `date_trunc('day'...)` full-day range (correctly NOT tightened). |
| 4 | Solar AC → House stat (Panel 9) reads from CT tables | ✓ VERIFIED | Panel 9 refId=A: `MAX(power) FROM east_microinverter_power`, refId=B: `AVG(power) FROM west_microinverter_power`. Both at 10s window. |
| 5 | Solar AC Production time-series (Panel 12) reads from CT tables | ✓ VERIFIED | Panel 12 refId=East: `AVG(power) AS "East" FROM east_microinverter_power`, refId=West: `FROM west_microinverter_power`. Uses `$__timeFrom/$__timeTo` (correctly NOT snapshot). |
| 6 | Inverter AC Power bargauge (Panel 13) reads from CT tables | ✓ VERIFIED | Panel 13 refId=East: `AVG(power) AS "East" FROM east_microinverter_power`, refId=West: `FROM west_microinverter_power`. At 10s window. |
| 7 | Power Profile chart (Panel 801) Production series reads from CT tables | ✓ VERIFIED | Panel 801 refId=Production: UNION ALL subquery with `AVG(power) AS pw FROM east_microinverter_power` + `west_microinverter_power`. Uses `$__timeFrom/$__timeTo`. |
| 8 | Calculated Load stat (Panel 905) East/West targets read from CT tables | ✓ VERIFIED | Panel 905 refId=East: `east_w FROM east_microinverter_power`, refId=West: `west_w FROM west_microinverter_power`. Grid target also at 10s. |
| 9 | All CT-table snapshot panels use 10-second recency window instead of 1-minute | ✓ VERIFIED | Full scan of all panels: zero CT-table targets found with `INTERVAL '1 minutes'` and `now()`. All 28 CT snapshot targets across 15 panels confirmed at `INTERVAL '10 seconds'`. |
| 10 | Dashboard refresh interval is 5s | ✓ VERIFIED | `"refresh": "5s"` at dashboard root. `"version": 47`. |
| 11 | Panels that stay on inverter (DC channels, temperature, state, today_production, total_production) are unchanged | ✓ VERIFIED | Panels 23-31 still at `INTERVAL '1 minutes'` on `"East/West Microinverter"`. Panels 2, 22, 45, 46, 24, 25 all still reference `Microinverter` tables. 51 inverter targets confirmed unchanged. |

**Score:** 11/11 truths verified (automated)

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `solar-pv-monitor.json` | Updated dashboard with 8 panels migrated, 15 panels with tightened windows, 5s refresh | ✓ VERIFIED | Valid JSON, 8 refs each to east/west_microinverter_power, `refresh: "5s"`, `version: 47`. 30 refs each to `East/West Microinverter` remain for unmigrated panels. No `power_sensor` in any CT-table query. |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| Panel 1 (Solar AC Power) | east/west_microinverter_power | `AVG(power) FROM east_microinverter_power` | ✓ WIRED | Pattern `AVG\(power\).*FROM east_microinverter_power` matched in refId=A. refId=B uses west table. |
| Panel 6/905 (Self-Consumption, Calculated Load) | CT tables for East/West, grid | `east_w FROM east_microinverter_power` | ✓ WIRED | Both panels 6 and 905 have East/West targets on CT tables, Grid target on `"grid"`. Expression math targets ($Solar, $Total, $Self) reference these refIds. |
| Panel 801 (Power Profile) Production | east/west_microinverter_power via UNION ALL | `AVG(power) AS pw FROM east_microinverter_power ... UNION ALL ... west_microinverter_power` | ✓ WIRED | Full UNION ALL subquery confirmed with both CT tables. |
| Pre-existing CT panels (4, 5, 10, 11, 16, 19, 902-906) | grid / 235_floor_1 / 235_floor_2 / 236_floor_1 | `INTERVAL '10 seconds'` | ✓ WIRED | All 10 pre-existing CT panels (16 targets) confirmed at 10-second window. Zero CT targets remain at 1 minute. |

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|--------------------|--------|
| `solar-pv-monitor.json` | SQL queries in rawSql | InfluxDB 3 Core (live CT sensors) | ? Cannot verify without running Grafana | ? NEEDS HUMAN — CT tables confirmed live since 2026-04-08T23:26:45 per context, but actual data flow requires Grafana import |

### Behavioral Spot-Checks

Step 7b: SKIPPED — This is a Grafana dashboard JSON file. It cannot be executed standalone; it requires import into a running Grafana instance connected to InfluxDB. Live data verification is routed to human verification (Task 2 in plan).

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| OVER-01 | 11-01 | Current total solar production power (kW) as large stat | ✓ SATISFIED | Panel 1 now reads from CT tables (faster, more accurate power readings) |
| OVER-05 | 11-01 | Current house load (kW) = solar + grid | ✓ SATISFIED | Panel 5 tightened to 10s window; Panel 6 Solar component migrated to CT |
| OVER-06 | 11-01 | Self-consumption ratio (%) | ✓ SATISFIED | Panel 6 East/West targets migrated to CT; expression math unchanged |
| OVER-09 | 11-01 | Peak power achieved today (kW) | ✓ SATISFIED | Panel 7 reads MAX(power) from CT tables over full day range |
| PROD-01 | 11-01 | Production time-series graph | ✓ SATISFIED | Panel 12 reads from CT tables; denser data points at 2s intervals |
| PROD-02 | 11-01 | Per-inverter production breakdown (East vs West) | ✓ SATISFIED | Panel 13 bargauge reads from CT tables at 10s window |
| GRID-01 | 11-01 | Grid voltage and frequency stats | ✓ SATISFIED | Panel 16 (voltage) tightened to 10s window; Grid current (Panel 19) also at 10s |

No orphaned requirements found — all 7 requirement IDs from the plan are accounted for and mapped to this phase's changes.

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| — | — | — | — | No anti-patterns found |

No TODOs, FIXMEs, placeholders, or stubs detected. No CT-table queries reference `power_sensor`. No empty `rawSql` in migrated panels. JSON validates cleanly.

### Human Verification Required

### 1. Live CT Data Flow in Grafana

**Test:** Import `solar-pv-monitor.json` into Grafana (or reload via Dashboard Settings → JSON Model). During daytime solar production, verify all 8 migrated panels display real-time CT power readings.
**Expected:** Panel 1 shows positive watt value updating every 5s. Panels 12 and 801 show smooth production curves. Panel 7 shows day's peak. Panel 6 shows valid self-consumption %. Panel 905 shows calculated load. Values should be smoother and more responsive than the old 60s inverter polling.
**Why human:** Cannot verify live InfluxDB data flow or visual rendering programmatically — requires Grafana UI with active CT sensor data.

### 2. Nighttime Behavior

**Test:** During nighttime, verify solar panels show 0 or "No data" (not frozen stale values).
**Expected:** 10-second recency window means stale daytime values drop off within 10 seconds of last reading. Panels should not show "last known" production during night.
**Why human:** Requires observing dashboard at night when inverters are offline.

### 3. Unchanged Panels Still Working

**Test:** Verify DC channel panels (22, 45), temperature gauges (24, 25), device state panels (26-31), and energy counter panels (2, 14, 15, 38-41) still display correctly.
**Expected:** All unchanged panels show their normal data at their normal refresh rates.
**Why human:** Cannot verify visual rendering or data correctness without live Grafana instance.

### Gaps Summary

No automated gaps found. All 11 observable truths verified at the code/JSON level:
- 8 panels successfully migrated from inverter tables to CT sensor tables
- All 28 CT snapshot targets across 15 panels tightened to 10-second recency windows
- Dashboard refresh changed to 5s, version bumped to 47
- Inverter-only panels verified unchanged

The only remaining verification is human confirmation that live CT data flows correctly through Grafana — this is Task 2 (human checkpoint) in the plan, which is expected and by design.

---

_Verified: 2026-04-09T00:30:00Z_
_Verifier: the agent (gsd-verifier)_
