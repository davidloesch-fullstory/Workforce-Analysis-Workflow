# Metric Ledger: {WFA-OrgName-YYYY-MM}

Org: {org_id}
Date Range: {start} – {end}
Teams: {team1}, {team2}, ...
Created: {timestamp}

## Metrics

| Metric ID | Query | Output Type | Signal Name | Blueprint Role | Significance | Baseline Value |
|-----------|-------|-------------|-------------|----------------|--------------|----------------|
| | | | | | | |

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
