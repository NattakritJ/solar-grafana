# Phase 5: Fix Transformation-Based Calculations → Expressions - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-01
**Phase:** 05-fix-all-panel-in-dashboard-that-still-use-transformation-to-calcuate-to-use-expression-instead
**Areas discussed:** Panels 5 & 6 fix approach, Scope (which panels to migrate), Solar Power sum pattern, Financial panels, Merge-only panels, Panel 10

---

## Which panels to discuss

| Option | Description | Selected |
|--------|-------------|----------|
| Expression content for panels 5 & 6 | Fill blank Expression targets | |
| Scope: which panels to migrate | Calculation-only vs all 21 panels | |
| Merge-only panels | bargauge/table/canvas with merge | |
| Handling broken Expression targets | Primary bug to fix | |

**User's choice:** "You decide but mainly the result of this phase is to reduce usage of transformation if it's using for calculation and use math expression instead (calculate on database)"

---

## Panels 5 & 6 Fix Approach

| Option | Description | Selected |
|--------|-------------|----------|
| Fix the blank Expression targets | Fill in blank math expressions in existing Expression targets | ✓ |
| Fix + clean disabled transforms | Remove old disabled calculateField transforms, clean up, then set correct expr values | |
| Rebuild panels 5 & 6 | Rebuild panels from scratch with clean Expression-based setup | |

**User's choice:** Fix the blank Expression targets
**Notes:** Minimal change — complete what's already set up rather than rebuilding.

---

## Scope: Which panels to migrate

| Option | Description | Selected |
|--------|-------------|----------|
| Calculation transforms only | Only fix panels where transformations are used for calculation | ✓ |
| All 21 panels | Fix everything with any transformation — make all 21 panels transformation-free | |
| Only the broken panels first | Fix only the three broken panels (5, 6, 10) with empty Expression targets | |

**User's choice:** Calculation transforms only
**Notes:** Panels using merge for display (table, bargauge, canvas) are out of scope.

---

## Solar Power sum pattern (panels 1, 2, 3, 7, 9)

| Option | Description | Selected |
|--------|-------------|----------|
| Keep merge+reduce | Keep as-is — reduce(sum) on two series is not brittle | |
| Migrate to Expression target | Use $A + $B math target | ✓ |
| Single SQL query | Consolidate into a single SQL query | |

**User's choice:** Migrate to Expression target
**Notes:** Consistent with the rest of the expression migration.

---

## Financial panels (panels 38-44)

| Option | Description | Selected |
|--------|-------------|----------|
| Migrate to Expression ($A + $B) | Sum East and West TOU savings via Expression | ✓ |
| Keep merge+reduce as-is | TOU SQL is complex enough | |
| Single consolidated SQL query | UNION or combined SELECT | |

**User's choice:** Migrate to Expression ($A + $B)

---

## Merge-only panels (13, 14, 15, 22, 23, 45)

| Option | Description | Selected |
|--------|-------------|----------|
| Out of scope — merge is fine for display | Leave with merge | ✓ |
| In scope — eliminate merge too | Remove merge and restructure | |

**User's choice:** Out of scope — merge is fine for display

---

## Panel 10 (duplicate House Load)

| Option | Description | Selected |
|--------|-------------|----------|
| Fix panel 10 same as panel 5 | Same Expression fix | ✓ |
| Fix panel 5 only | Investigate panel 10 before touching | |
| Remove panel 10 as duplicate | Delete if truly a duplicate | |

**User's choice:** Fix panel 10 same as panel 5

---

## Agent's Discretion

- Exact Expression target JSON structure
- Whether to hide raw InfluxDB query targets after adding expressions
- Order of targets array in panel JSON
- Whether `reduceOptions` needs updating after switching from reduce to Expression

## Deferred Ideas

None.
