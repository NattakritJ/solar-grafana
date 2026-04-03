---
status: resolved
trigger: "Today's Savings panel shows 76.05 THB at 3AM when solar panels are not producing any power"
created: 2026-04-04T03:00:00Z
updated: 2026-04-04T03:30:00Z
---

## Current Focus
<!-- OVERWRITE on each update - reflects NOW -->

hypothesis: CONFIRMED AND FIXED — Replaced all 14 occurrences of "now() AT TIME ZONE 'Asia/Bangkok'" with "tz(now(), 'Asia/Bangkok')" in solar-pv-monitor.json. JSON validated successfully.
test: User to import updated dashboard into Grafana and verify at night: Today's Savings = 0, Today's Energy = 0, Peak Today = 0; during daytime these should show accumulating values.
expecting: At 3AM, Today's Savings shows 0 THB (not yesterday's 76.05). Daytime values still accumulate correctly.
next_action: AWAITING USER VERIFICATION

## Symptoms
<!-- Written during gathering, then IMMUTABLE -->

expected: Today's Savings panel should show 0 THB at 3AM (or correct accumulated savings from solar production during the day)
actual: Panel shows 76.05 THB at 3AM when there is no solar production
errors: No error messages — panel displays a value that appears incorrect for the current time
reproduction: View the dashboard at night (3AM) — "Today's Savings" stat panel shows a non-zero THB value
started: Noticed at 3AM; unclear if always wrong at night or just tonight

## Eliminated
<!-- APPEND only - prevents re-investigating -->

- hypothesis: The value 76.05 THB is legitimate "today's accumulated savings" from daytime production (correct behavior)
  evidence: If it were correct behavior, the query would anchor to today midnight Bangkok. Traced the AT TIME ZONE semantics in InfluxDB 3 docs: now() has no TZ, AT TIME ZONE on a no-TZ timestamp is a WALL CLOCK label (not a conversion). So 20:15 UTC is labeled as "20:15 Bangkok" not converted to Bangkok 03:15. The query actually anchors to YESTERDAY midnight Bangkok, showing yesterday's savings.
  timestamp: 2026-04-04T03:20:00Z

- hypothesis: Query uses $__timeFrom/$__timeTo dashboard variables (wrong time range set)
  evidence: Query uses now() directly in SQL with date_trunc — no dashboard time range variables used. The bug is in the SQL timezone logic, not the Grafana time range picker.
  timestamp: 2026-04-04T03:10:00Z

- hypothesis: Stale Grafana cache (browser tab in background)
  evidence: The query uses now() which re-evaluates on each refresh. Even if the page was live at 3AM, the date_trunc timezone bug would produce the wrong anchor date regardless of caching.
  timestamp: 2026-04-04T03:25:00Z

## Evidence
<!-- APPEND only - facts discovered -->

- timestamp: 2026-04-04T03:10:00Z
  checked: solar-pv-monitor.json panel id=38 (Today's Savings), rawSql targets A and B
  found: Query uses date_trunc('day', now() AT TIME ZONE 'Asia/Bangkok') AT TIME ZONE 'UTC' as the time anchor
  implication: This pattern is the source of the bug

- timestamp: 2026-04-04T03:15:00Z
  checked: InfluxDB 3 Core SQL docs — AT TIME ZONE vs tz() behavior
  found: "When using an input timestamp that does not have a timezone (the default behavior in InfluxDB) with the AT TIME ZONE operator, the operator returns the same timestamp, but with a timezone offset (also known as the wall clock time)". now() returns a no-TZ UTC timestamp. AT TIME ZONE 'Asia/Bangkok' just labels it as +07:00 without converting.
  implication: now() AT TIME ZONE 'Bangkok' treats the UTC hour value as the Bangkok hour — OFF BY 7 HOURS. At 3AM Bangkok (20:15 UTC), the query thinks Bangkok time is 20:15. It calculates today Bangkok midnight as April 3 00:00+07:00 = April 2 17:00 UTC. This is YESTERDAY midnight Bangkok, not today midnight.

- timestamp: 2026-04-04T03:20:00Z
  checked: All occurrences of now() AT TIME ZONE 'Asia/Bangkok' in the dashboard
  found: 14 instances across: Today's Energy (2), Peak Today (2), Today's Savings (2), Month's Savings (2), Year's Savings (2), Grid Export Count (1), Grid Max Export (1), and 2 others
  implication: All "today" date-anchored queries are off by 7 hours. Every query that computes a daily/monthly/yearly aggregate using this pattern shows data shifted by a full day (anchoring to yesterday instead of today).

- timestamp: 2026-04-04T03:22:00Z
  checked: InfluxDB 3 tz() function docs
  found: "tz() always converts the input timestamp to the specified time zone. If the input timestamp does not have a timezone, the function assumes it is a UTC timestamp." tz(now(), 'Asia/Bangkok') correctly converts UTC 20:15 -> Bangkok 03:15+07:00. Then date_trunc('day', ...) gives midnight Bangkok April 4 = UTC April 3 17:00. This is TODAY midnight Bangkok.
  implication: Replacing now() AT TIME ZONE 'Asia/Bangkok' with tz(now(), 'Asia/Bangkok') fixes the 7-hour anchor error for all 14 queries.

## Resolution
<!-- OVERWRITE as understanding evolves -->

root_cause: InfluxDB 3 SQL's AT TIME ZONE operator has different semantics depending on whether the input timestamp carries a timezone. now() returns UTC time WITHOUT a timezone marker. AT TIME ZONE 'Asia/Bangkok' on a no-TZ timestamp is a "wall clock" label operation — it just attaches the +07:00 offset without converting the value. So now() AT TIME ZONE 'Bangkok' at 20:15 UTC produces "20:15+07:00" (treating 20:15 as Bangkok time), NOT "03:15+07:00" (the correct UTC→Bangkok conversion). This makes date_trunc('day', ...) compute yesterday's Bangkok midnight instead of today's, causing all 14 daily/monthly/yearly aggregate queries to anchor 7 hours too early and include data from the previous Bangkok day.
fix: Replaced all 14 occurrences of "now() AT TIME ZONE 'Asia/Bangkok'" with "tz(now(), 'Asia/Bangkok')" in solar-pv-monitor.json. The tz() function always treats a no-TZ timestamp as UTC and correctly converts it to the target timezone. JSON validated after replacement.
verification: Confirmed fixed by user in live Grafana — Today's Savings correctly shows 0 THB at 3AM after replacing tz() function.
files_changed: [solar-pv-monitor.json]
