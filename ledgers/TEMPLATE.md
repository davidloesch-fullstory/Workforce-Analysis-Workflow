# Metric Ledger: {WFA-OrgName-YYYY-MM}

Org: {org_id}
Analysis Window: {analysis_start} – {analysis_end}  (absolute, pinned in Phase 0.0)
Computed As-Of: {as_of}  (timestamp + timezone)
Metric URL Format: {verified_url_format}  (from Phase 0.0.3 — note if query params needed)
Teams: {team1} ({people-scoped|tool-cluster}), {team2} (...), ...
Created: {timestamp}

## Segments (people-scoped teams)

One row per team that maps to a user-property value (Phase 0.4.2). Tool-cluster
teams have no segment. Segments are pinned to the Analysis Window.

| Segment ID | Team | Definition (property = value) | Window | Scoping Mode |
|------------|------|-------------------------------|--------|--------------|
| | | | | |

## Metrics

Every metric is built with absolute `start_date`/`end_date` equal to the Analysis
Window above. The **Window** column records the exact dates used per metric; any
row that differs from the Analysis Window means a relative preset leaked in and
must be rebuilt. **Segment ID** records the team segment a metric was scoped to
(blank = org-wide context or a tool-cluster team's URL-scoped metric). **Time
Property** records which time measure a metric uses — `focused` (`Page focused
time (new)`, the primary engaged-time measure), `active` (`Page active time
(new)`, primary tool only, for the work-type ratio), or `N/A` (non-time metric).

| Metric ID | Query | Output Type | Window | Segment ID | Time Property | Signal Name | Blueprint Role | Significance | Baseline Value |
|-----------|-------|-------------|--------|------------|---------------|-------------|----------------|--------------|----------------|
| | | | | | | | | | |

## How to Use This Ledger

1. **During Phase 1**: Append a row after each `build_metric` → `compute_metric` pair.
   Set Blueprint Role and Significance to "TBD".

2. **During Phase 3**: Backfill Blueprint Role and Significance for every row.
   Tag each metric as "Primary signal for #{rank} {name}" or "Baseline context".

3. **During Phase 4**: Use this ledger to populate the Measurement & Proof tab
   in the HTML deliverable. Each row becomes a row in the Baseline Metrics table.

4. **During Phase 5**: Re-compute each metric with post-implementation date ranges.
   Compare to Baseline Value to calculate deltas.

## Suggested Dashboard Names

After backfilling, each metric gets a suggested FullStory dashboard name:
`[{prefix}] {Signal Name}` — e.g., `[WFA-Acme-2026-05] Copied Comment Log`

Use this naming convention when saving metrics in FullStory for dashboard use.
