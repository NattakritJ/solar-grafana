---
phase: 06-financial-savings-rework-use-fixed-rate-3-5-thb-kwh-instead-of-tou-rate
verified: 2026-04-01T21:30:00Z
status: passed
score: 5/5 must-haves verified
re_verification: false
---

# Phase 06: Financial Savings Rework — Flat-Rate 3.5 THB/kWh Verification Report

**Phase Goal:** Replace the complex TOU peak/off-peak rate calculation with a single flat rate of 3.5 THB/kWh for all savings panels, and remove the now-irrelevant TOU breakdown panels (Peak kWh, Peak THB, Off-Peak kWh, Off-Peak THB)
**Verified:** 2026-04-01T21:30:00Z
**Status:** ✅ PASSED
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Today's Savings panel (38) shows THB calculated at 3.5 THB/kWh flat rate (not TOU) | ✓ VERIFIED | `rawSql` A+B both contain `SUM(hourly_kwh * 3.5)`; no `CASE WHEN`, `5.7982`, or `2.6369` |
| 2 | Month Savings panel (39) shows THB calculated at 3.5 THB/kWh flat rate | ✓ VERIFIED | Same SQL pattern with `date_trunc('month', now() AT TIME ZONE 'Asia/Bangkok') AT TIME ZONE 'UTC'` boundary |
| 3 | Year Savings panel (40) shows THB calculated at 3.5 THB/kWh flat rate | ✓ VERIFIED | Same SQL pattern with `date_trunc('year', now() AT TIME ZONE 'Asia/Bangkok') AT TIME ZONE 'UTC'` boundary |
| 4 | The 4 TOU-specific panels (41/42/43/44) no longer exist in the dashboard | ✓ VERIFIED | Panel IDs confirmed absent from `solar-pv-monitor.json` panels array (total 47 panels, none are 41–44) |
| 5 | Financial Savings row contains exactly 3 stat panels: Today's Savings, Month Savings, Year Savings | ✓ VERIFIED | Row 600 at y=84; panels 38 (x=0,w=8), 39 (x=8,w=8), 40 (x=16,w=8) fill the row cleanly at y=85 |

**Score:** 5/5 truths verified

---

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `solar-pv-monitor.json` | Updated dashboard JSON with simplified flat-rate savings SQL and TOU panels removed; contains `hourly_kwh * 3.5` | ✓ VERIFIED | File exists, valid JSON (`schemaVersion: 42`, title: "Solar PV Monitor"), contains pattern; commits `d7ac58c` (flat rate) + `f50a84f` (Bangkok TZ fix) |

---

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| Panel 38 refId=A rawSql | InfluxDB `"East Microinverter"` | `SUM(hourly_kwh * 3.5)` with today boundary | ✓ WIRED | Full SQL confirmed: `WHERE time >= date_trunc('day', now() AT TIME ZONE 'Asia/Bangkok') AT TIME ZONE 'UTC'` |
| Panel 38 refId=B rawSql | InfluxDB `"West Microinverter"` | `SUM(hourly_kwh * 3.5)` with today boundary | ✓ WIRED | Same SQL structure, West measurement |
| Panel 39 refId=A rawSql | InfluxDB `"East Microinverter"` | `SUM(hourly_kwh * 3.5)` with month boundary | ✓ WIRED | `WHERE time >= date_trunc('month', now() AT TIME ZONE 'Asia/Bangkok') AT TIME ZONE 'UTC'` |
| Panel 39 refId=B rawSql | InfluxDB `"West Microinverter"` | `SUM(hourly_kwh * 3.5)` with month boundary | ✓ WIRED | Same SQL structure, West measurement |
| Panel 40 refId=A rawSql | InfluxDB `"East Microinverter"` | `SUM(hourly_kwh * 3.5)` with year boundary | ✓ WIRED | `WHERE time >= date_trunc('year', now() AT TIME ZONE 'Asia/Bangkok') AT TIME ZONE 'UTC'` |
| Panel 40 refId=B rawSql | InfluxDB `"West Microinverter"` | `SUM(hourly_kwh * 3.5)` with year boundary | ✓ WIRED | Same SQL structure, West measurement |
| Panels 38/39/40 refId=C | East + West totals | Expression `$A + $B` | ✓ WIRED | All three panels retain their Expression (math) target unchanged |

---

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|--------------------|--------|
| Panel 38 (Today's Savings) | `savings_thb` via `$A + $B` expression | InfluxDB `"East Microinverter"` + `"West Microinverter"`, bucketed hourly | Yes — real aggregate query against live measurements; no static fallback | ✓ FLOWING |
| Panel 39 (Month Savings) | `savings_thb` via `$A + $B` expression | Same as above, month boundary | Yes — same pattern, different time window | ✓ FLOWING |
| Panel 40 (Year Savings) | `savings_thb` via `$A + $B` expression | Same as above, year boundary | Yes — same pattern, different time window | ✓ FLOWING |

**Note:** `WHERE hourly_kwh > 0` filter correctly excludes night-time zero-production buckets from the savings sum.

---

### Behavioral Spot-Checks

Step 7b: SKIPPED (no runnable entry point — dashboard JSON is imported into Grafana, not a locally runnable server). Human verification substitutes for behavioral checks (see below).

---

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|-------------|-------------|--------|----------|
| FINC-01 | 06-01-PLAN.md | User can see today's estimated savings in THB based on solar production | ✓ SATISFIED | Panel 38 "Today's Savings" verified with flat-rate SQL producing THB output |
| FINC-02 | 06-01-PLAN.md | User can see TOU-aware savings calculations (5.7982 peak / 2.6369 off-peak) | ✓ SUPERSEDED | Phase 06 intentionally replaces TOU with flat 3.5 THB/kWh — homeowner's actual tariff is flat rate; FINC-02 text is now obsolete; flat rate is the correct behaviour per confirmed decision |
| FINC-03 | 06-01-PLAN.md | User can see cumulative savings for month-to-date and year-to-date periods | ✓ SATISFIED | Panels 39 (Month) and 40 (Year) verified with correct Bangkok-timezone boundaries |

#### Note on FINC-02 and FINC-04 (orphaned from REQUIREMENTS.md)

REQUIREMENTS.md contains **FINC-02** (TOU-aware calculation) and **FINC-04** (peak/off-peak breakdown) mapped to Phase 4 (Complete). Phase 06 intentionally supersedes these: the homeowner's electricity tariff is flat rate, not TOU — the original TOU implementation was calculating *incorrect* savings. Phase 06's PLAN explicitly lists only FINC-01, FINC-02, FINC-03 as its requirements.

- **FINC-02**: Its Phase 4 implementation (TOU CASE WHEN SQL) has been deliberately replaced. The requirement text is stale. The correct behaviour is now flat-rate savings. **Not a gap — an intentional supersession.**
- **FINC-04**: Panels 41–44 (Peak kWh, Peak THB, Off-Peak kWh, Off-Peak THB) were removed. With flat rate, peak/off-peak breakdown is meaningless. **Not a gap — intentionally removed.** REQUIREMENTS.md traceability table still maps this to Phase 4 and does not claim Phase 6 coverage; the omission is correct.

**Recommendation:** Update REQUIREMENTS.md to mark FINC-02 as superseded and FINC-04 as deliberately removed, and add a note under the Financial section documenting the flat-rate decision.

---

### Grid Position Verification

| Panel | Expected Position | Actual Position | Status |
|-------|-------------------|-----------------|--------|
| Row 600 (Financial Savings) | y=84 | y=84, w=24, h=1 | ✓ |
| Panel 38 (Today's Savings) | y=85, x=0, w=8, h=4 | y=85, x=0, w=8, h=4 | ✓ |
| Panel 39 (Month Savings) | y=85, x=8, w=8, h=4 | y=85, x=8, w=8, h=4 | ✓ |
| Panel 40 (Year Savings) | y=85, x=16, w=8, h=4 | y=85, x=16, w=8, h=4 | ✓ |
| Row 700 (Backfeed Log) | y=89 | y=89, w=24, h=1 | ✓ |
| Panel 33 (Today's Backfeed Count) | y=90 (shifted up 4) | y=90 | ✓ |
| Panel 34 (Max Backfeed Power) | y=90 (shifted up 4) | y=90 | ✓ |
| Panel 35 (Grid Backfeed Log) | y=90 (shifted up 4) | y=90 | ✓ |

---

### Anti-Patterns Found

| File | Pattern | Severity | Impact |
|------|---------|----------|--------|
| `solar-pv-monitor.json` | None found | — | — |

Full scan: No `CASE WHEN`, no `5.7982`, no `2.6369` anywhere in panels 38/39/40 rawSql. No `TODO`/`FIXME`/placeholder markers. No empty implementations. No hardcoded empty data. Expression targets `$A + $B` intact on all three savings panels. Dashboard JSON structurally valid.

---

### Human Verification

**Status: APPROVED** (per user confirmation at Task 2 checkpoint)

1. **Flat-rate savings display in Grafana**
   - **Tested:** Imported `solar-pv-monitor.json` into Grafana; viewed Financial Savings row
   - **Result:** Today's Savings, Month Savings, Year Savings all display plausible flat-rate THB values; Peak kWh / Peak THB / Off-Peak kWh / Off-Peak THB panels are absent; Backfeed Log row remains intact below
   - **Approved:** ✅

---

### Commits

| Commit | Type | Description |
|--------|------|-------------|
| `d7ac58c` | feat | Replace TOU SQL with flat-rate 3.5 THB/kWh, remove panels 41-44 |
| `f50a84f` | fix | Apply Bangkok timezone to month/year savings boundaries (auto-fixed deviation) |
| `97f844b` | docs | Complete financial savings rework plan — flat-rate 3.5 THB/kWh verified |

---

## Summary

Phase 06 goal is **fully achieved**. The codebase evidence matches every claimed change in the SUMMARY:

- **Flat rate implemented:** All 6 rawSql targets (A+B on panels 38, 39, 40) use `SUM(hourly_kwh * 3.5) AS savings_thb` — zero TOU logic remaining
- **TOU panels removed:** IDs 41, 42, 43, 44 are absent from the dashboard JSON
- **Bangkok timezone correct:** All three boundaries use the correct `date_trunc('period', now() AT TIME ZONE 'Asia/Bangkok') AT TIME ZONE 'UTC'` pattern (today/month/year), including the deviation fix in `f50a84f`
- **Expression targets preserved:** All three panels retain `$A + $B` math expressions for East+West summation
- **Grid layout correct:** Financial Savings row holds exactly 3 panels in 8+8+8 layout; Backfeed Log row shifted to y=89 with panels at y=90
- **JSON validity:** `solar-pv-monitor.json` is valid JSON, schemaVersion 42, 47 panels total
- **Human verified:** User confirmed correct flat-rate values displayed in Grafana with no TOU panels visible

---

_Verified: 2026-04-01T21:30:00Z_
_Verifier: the agent (gsd-verifier)_
