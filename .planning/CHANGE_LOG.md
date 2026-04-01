# Dashboard Change Log

This file records manual edits applied directly to `solar-pv-monitor.json` outside of formal GSD phase plans.

---

## 2026-04-02 — Post-v1 JSON Cosmetic & System Status Polish

**Commit context:** Manual edits applied after Phase 07 completion.  
**Dashboard version:** bumped 20 → 27 (Grafana increments on each save/import).

### Changes

#### 1. System Status panel (panel id=8) — display options rework

| Property | Before | After | Why |
|---|---|---|---|
| `colorMode` | `"value"` | `"none"` | Remove per-value colour tinting; text-only display is cleaner for a multi-field state panel |
| `justifyMode` | `"auto"` | `"center"` | Center-align the inverter state labels in the stat cell |
| `textMode` | `"value"` | `"value_and_name"` | Show both the field name and its value so the viewer knows which inverter each status refers to |
| `wideLayout` | `true` | `false` | Compact layout fits the two-inverter state values on one line |

#### 2. `transparent: true` removed from three Power Flow panels

Panels affected:
- **☀ Solar → House** (panel id in Power Flow row)
- **🏠 House Load**
- **⚡ Grid → House**

These panels now render with the standard panel background. Matches the convention established in the Post-v1 STATE.md entry (2026-04-01) where transparent was removed from panels 6 and 8 because they moved to dedicated rows.

#### 3. JSON formatting — single-line arrays (cosmetic only, no logic change)

31 occurrences of multi-line single-element arrays like:
```json
"calcs": [
  "lastNotNull"
]
```
were collapsed to:
```json
"calcs": ["lastNotNull"]
```
Also collapsed: `"displayLabels"`, `"reducers"`, `"tags"`, and legend `calcs` arrays elsewhere in the file. **No functional change** — this is a Grafana export formatting difference.

---

## Decision Reference

See `STATE.md → Accumulated Context → Post-v1 Edits Applied` for the full history of post-milestone manual changes.
