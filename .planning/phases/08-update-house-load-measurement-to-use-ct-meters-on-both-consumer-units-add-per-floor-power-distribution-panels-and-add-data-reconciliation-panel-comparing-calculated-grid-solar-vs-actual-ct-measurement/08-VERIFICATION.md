---
phase: 08-update-house-load-measurement-to-use-ct-meters-on-both-consumer-units-add-per-floor-power-distribution-panels-and-add-data-reconciliation-panel-comparing-calculated-grid-solar-vs-actual-ct-measurement
verified: 2026-04-02T14:45:30Z
status: human_needed
score: 8/8 must-haves verified
re_verification: false
human_verification:
  - test: "Import solar-pv-monitor.json into Grafana 12.4.1 and verify House Load panels show live CT values"
    expected: "Panel 5 (House Load stat) and Panel 10 (🏠 House Load power flow) show a reasonable value (~1,500–2,000W) drawn from CT meters, not the old derived formula"
    why_human: "Cannot verify live InfluxDB query returns against 235_floor_1 / 235_floor_2 measurements without a running Grafana+InfluxDB instance; also verifies the CT data pipeline from Phase 7 is functioning"
  - test: "Verify Power Distribution row renders all 4 panels with orange background colors and live CT values"
    expected: "Floor 1 Power (~1,000–1,500W orange), Floor 2 Power (~500–1,000W orange), CT Load (sum matching House Load), Calculated Load (within ~10% of CT Load)"
    why_human: "Visual appearance and live value comparison between CT Load and Calculated Load cannot be verified programmatically; color rendering depends on threshold ranges matching actual wattage"
  - test: "Verify Power Profile time-series shows 235 Floor 1 and 235 Floor 2 series"
    expected: "Two new CT series lines appear in the Power Profile chart for time ranges from 2026-04-02 onwards; historical dates correctly show no CT data (expected — CT data collection started 2026-04-02)"
    why_human: "Time-series rendering and series legend display require visual inspection in Grafana; data availability depends on CT collection pipeline being active since 2026-04-02"
  - test: "Scroll through entire dashboard to confirm no layout overlap or missing panels"
    expected: "All rows (Overview, Power Distribution, Power Profile, Production, Module Level, Inverter Health, Financial, Backfeed Log) display correctly with no overlap or hidden panels after the 7-unit y-shift"
    why_human: "Layout correctness and panel overlap require visual inspection; 37 downstream panels were shifted and visual confirmation of no clipping/overlap is necessary"
---

# Phase 8: CT Meter Integration & Power Distribution — Verification Report

**Phase Goal:** Replace derived House Load calculation with CT direct measurement, surface per-floor power distribution, and add reconciliation panels so the homeowner can compare CT-measured vs calculated load  
**Verified:** 2026-04-02T14:45:30Z  
**Status:** human_needed  
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Panel 5 (House Load stat) shows CT-measured load (floor1+floor2) not the old derived formula | ✓ VERIFIED | targets: `['Floor1','Floor2','Total']`; no `East/West/Grid` refs; rawSql contains `235_floor_1`/`235_floor_2`; expression `$Floor1 + $Floor2`; description updated to "CT direct measurement: Floor 1 + Floor 2" |
| 2 | Panel 10 (🏠 House Load power flow) shows CT-measured load (floor1+floor2) | ✓ VERIFIED | targets: `['Floor1','Floor2','Total']`; old `East/West/Grid` refs absent; identical CT replacement applied; description: "Total household power consumption (CT direct measurement)" |
| 3 | Dashboard has a new 'Power Distribution' row (row 900) between Overview and Power Profile | ✓ VERIFIED | Panel id=900, type=`row`, title=`"Power Distribution"`, gridPos.y=12; row 800 (Power Profile) is at y=19, confirming row 900 is sandwiched correctly |
| 4 | Power Distribution row contains: Floor 1 Power, Floor 2 Power, CT Load, Calculated Load stat panels — all 4 visible | ✓ VERIFIED | Panel 902 `"🏠 Floor 1 Power"` (x=0), 903 `"🏠 Floor 2 Power"` (x=6), 904 `"⚡ CT Load"` (x=12), 905 `"🔢 Calculated Load"` (x=18); all y=13, w=6, h=6; fills full 24-column row |
| 5 | Floor 1 and Floor 2 stat panels use orange color convention (consumption = orange) | ✓ VERIFIED | Panels 902 & 903 thresholds: `['dark-gray','light-orange','orange','dark-orange']`; colorMode=`background` |
| 6 | All CT queries use 2-minute recency window (WHERE time >= now() - INTERVAL '2 minutes') matching Phase 7 pattern | ✓ VERIFIED | 8 CT rawSql queries across panels 5, 10, 902, 903, 904 all contain `INTERVAL '2 minutes'`; verified individually |
| 7 | Power Profile row 800 and all downstream panels shifted down 7 units (still visible, no overlap) | ✓ VERIFIED | All 35 shifted panels match expected y values (e.g., 800→y=19, 801→y=20, 200→y=32 … 35→y=103); no overlap detected in layout scan |
| 8 | Panel 801 (Power Profile) shows Floor 1 and Floor 2 as additional time-series alongside Production/Grid/Load | ✓ VERIFIED | targets: `['Production','Grid','Load','Floor1','Floor2']`; Floor1 rawSql: `AVG(power)/1000.0 AS "235 Floor 1" FROM "235_floor_1"`; Floor2 identical for `235_floor_2`; both `format=time_series`, `rawQuery=true`, `$__timeFrom/$__timeTo` macros present |

**Score:** 8/8 truths verified

---

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `solar-pv-monitor.json` | Updated dashboard with CT meter integration, Power Distribution row, and updated Power Profile; contains `235_floor_1` | ✓ VERIFIED | File exists; 53 panels; version 45; `235_floor_1` appears 5 times, `235_floor_2` appears 5 times; valid JSON (parses without errors) |

---

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| Panel 5 targets | 235_floor_1 / 235_floor_2 measurements | Floor1/Floor2 InfluxDB queries + Expression `$Floor1 + $Floor2` | ✓ WIRED | Both rawSql contain `235_floor_*` with `WHERE time >= now() - INTERVAL '2 minutes'`; expression wires Floor1+Floor2 into Total |
| Panel 10 targets | 235_floor_1 / 235_floor_2 measurements | Floor1/Floor2 InfluxDB queries + Expression `$Floor1 + $Floor2` | ✓ WIRED | Identical to Panel 5 replacement; both Floor1/Floor2 queries + math expression present |
| Row 900 panels (902–905) | 235_floor_1, 235_floor_2, East/West Microinverter, Smart Meter | Stat panels with 2-minute recency window queries | ✓ WIRED | 902→`235_floor_1`; 903→`235_floor_2`; 904→both CT measurements + expression; 905→East/West Microinverter + Smart Meter + expression; all reconciliation queries present |
| Panel 801 new targets | 235_floor_1 / 235_floor_2 measurements | DATE_BIN 5-minute time-series query | ✓ WIRED | `Floor1` refId: `AVG(power)/1000.0 AS "235 Floor 1" FROM "235_floor_1" WHERE time >= $__timeFrom AND time <= $__timeTo GROUP BY 1 ORDER BY 1`; Floor2 identical for floor_2; format=`time_series` |

---

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|--------------------|--------|
| Panel 5 (House Load stat) | Total (= Floor1 + Floor2) | `235_floor_1`, `235_floor_2` via `selector_last(power, time)` | Yes — live DB queries, not static returns; COALESCE handles nulls gracefully | ✓ FLOWING |
| Panel 10 (🏠 House Load) | Total (= Floor1 + Floor2) | Same as Panel 5 | Yes — identical real queries | ✓ FLOWING |
| Panel 902 (Floor 1 Power) | Floor 1 Power | `235_floor_1` via `selector_last(power, time)` | Yes — real DB query | ✓ FLOWING |
| Panel 903 (Floor 2 Power) | Floor 2 Power | `235_floor_2` via `selector_last(power, time)` | Yes — real DB query | ✓ FLOWING |
| Panel 904 (CT Load) | Total (= Floor1 + Floor2) | CT measurements via `selector_last(power, time)` on both floors | Yes — both hidden queries + expression | ✓ FLOWING |
| Panel 905 (Calculated Load) | Total (= East + West + Grid) | East/West Microinverter `power_sensor` + Smart Meter `power_sensor` | Yes — uses existing `power_sensor` field proven in prior phases | ✓ FLOWING |
| Panel 801 (Power Profile) | 235 Floor 1, 235 Floor 2 series | `AVG(power)/1000.0 FROM 235_floor_*` with `DATE_BIN 5-min`, `$__timeFrom/$__timeTo` | Yes — time-range-aware aggregated query; data available from 2026-04-02 onwards | ✓ FLOWING |

---

### Behavioral Spot-Checks

Step 7b: SKIPPED — no runnable entry points. `solar-pv-monitor.json` is a Grafana dashboard export file; behavioral verification requires a live Grafana 12.4.1 + InfluxDB 3 Core instance. Live CT data validation is delegated to Human Verification items.

---

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| OVER-05 | 08-01-PLAN.md | User can see current house load (kW) as solar production + grid import | ✓ SATISFIED | Panel 5 and Panel 10 now use CT Floor1+Floor2 direct measurement — a more accurate fulfillment of the requirement (CT hardware measurement replaces the software-derived formula). Panels 904/905 provide CT vs calculated reconciliation. |
| OVER-03 | 08-01-PLAN.md | User can see lifetime total energy production (kWh/MWh) as a stat | ⚠️ PLANNING ARTIFACT | Phase 8 makes no changes to any lifetime energy panel. Panel 702 (`"Lifetime Energy"`, y=1) was established in Phase 1 and is untouched. OVER-03 appears to have been incorrectly included in the Phase 8 PLAN's requirements list — it was already satisfied in Phase 1 and Phase 8 adds no new evidence for it. The requirement is not broken, but Phase 8 cannot claim independent credit for it. |

**Orphaned requirements check:** REQUIREMENTS.md traceability table maps all requirements to Phases 1–4. Phases 5–8 are post-v1 enhancements not reflected in the traceability table — this is expected (not an error). No orphaned requirements found that are mapped to Phase 8 in REQUIREMENTS.md.

---

### Anti-Patterns Found

| File | Pattern | Severity | Impact |
|------|---------|----------|--------|
| — | — | — | No anti-patterns detected |

Scan covered panels 5, 10, 800, 801, 900, 902, 903, 904, 905:
- No TODO/FIXME/placeholder text in any panel JSON
- No `return null` / `return []` / `return {}` stub patterns
- No hardcoded empty data arrays (`= []`, `= {}`) in query targets
- CT field `power` (not `power_sensor`) used consistently for `235_floor_*` measurements
- Panel 905 correctly uses `power_sensor` for East/West Microinverter and Smart Meter (consistent with prior phases)

---

### Human Verification Required

#### 1. Live CT Panel Values

**Test:** Import `solar-pv-monitor.json` into Grafana 12.4.1 and view the Overview row  
**Expected:** Panel 5 "House Load" shows a live wattage value (~1,500–2,000W typical household load) drawn from CT measurements; Panel 10 "🏠 House Load" shows the same CT-based value in the power flow arrangement  
**Why human:** Cannot programmatically verify that the `235_floor_1` and `235_floor_2` measurements in InfluxDB are actively receiving data from CT sensors and returning non-zero values to Grafana

#### 2. Power Distribution Row Visual & Values

**Test:** Scroll to the "Power Distribution" row (between Overview and Power Profile) and inspect all 4 panels  
**Expected:**
- "🏠 Floor 1 Power" — shows floor 1 wattage with orange background color
- "🏠 Floor 2 Power" — shows floor 2 wattage with orange background color
- "⚡ CT Load" — shows sum of Floor 1 + Floor 2 (should match Panel 5 House Load)
- "🔢 Calculated Load" — shows old derived value (East+West+Grid); comparison with CT Load should be within ~10%  

**Why human:** Live value comparison between CT and calculated load, orange background threshold appearance at real wattage levels, and visual layout confirmation require Grafana rendering

#### 3. Power Profile CT Series

**Test:** Set time range to "Last 1 hour" (or any range from 2026-04-02 onwards) and inspect the Power Profile time-series  
**Expected:** Two new series "235 Floor 1" and "235 Floor 2" appear in the chart legend and as plotted lines; setting time range to before 2026-04-02 correctly shows no CT data (expected, not a bug)  
**Why human:** Time-series legend rendering and series visibility require visual inspection in Grafana

#### 4. Full Dashboard Scroll — No Layout Overlap

**Test:** Scroll through the entire dashboard from top to bottom  
**Expected:** All rows visible in order: Overview → Power Distribution → Power Profile → Production → Module Level → Inverter Health → Financial → Backfeed Log; no panels overlapping, clipped, or hidden as a result of the 7-unit y-shift applied to 37 downstream panels  
**Why human:** Layout integrity with 37 shifted panels can only be confirmed visually in Grafana's rendering engine

---

### Gaps Summary

No structural gaps found. All 8 observable truths are verified against the actual `solar-pv-monitor.json` codebase. The dashboard JSON is structurally complete and correct.

**Note on OVER-03:** The PLAN lists OVER-03 as a requirement covered by Phase 8, but Phase 8 makes no changes relevant to lifetime energy tracking. This is a planning artifact (OVER-03 was satisfied in Phase 1 and remains satisfied). It does not constitute a gap — the requirement is met; only the phase attribution is incorrect.

Four human verification items remain — these are standard for a Grafana dashboard phase and cover visual rendering, live data, and layout confirmation that cannot be verified from the JSON alone.

---

*Verified: 2026-04-02T14:45:30Z*  
*Verifier: the agent (gsd-verifier)*
