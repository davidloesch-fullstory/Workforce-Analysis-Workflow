# Metric Ledger: WFA-FSWorkforce-2026-05

Org: 60BC (FS Workforce Internal)
Date Range: 2026-04-27 – 2026-05-26
Teams: Engineering
Created: 2026-05-26

| Metric ID | Query | Output Type | Signal Name | Blueprint Role | Significance | Baseline Value |
|-----------|-------|-------------|-------------|----------------|--------------|----------------|
| 2104125756 | page views grouped by top-level URL domain | top_n | Tool Inventory (by domain) | Baseline context | Frames overall tool usage landscape | 949,220 total PV |
| 1264724432 | page views grouped by URL path where URL domain contains github.com | top_n | GitHub Page Views (by path) | Supports #2 Auth Friction, #5 PR Navigation | Shows GitHub navigation patterns incl. SAML/OAuth paths | 49,944 total |
| 8347678 | page views grouped by URL path where URL domain contains fullstory.atlassian.net | top_n | Jira Page Views (by path) | Supports #3 Board Polling | Shows Jira board/backlog navigation patterns | 36,573 total |
| 437033618 | page views grouped by URL path where URL domain contains app.circleci.com | top_n | CircleCI Page Views (by path) | Primary signal for #1 Pipeline Polling | Directly measures CI status-checking behavior | 9,611 total |
| 1036604012 | page views grouped by URL path where URL domain contains fullstory.grafana.net | top_n | Grafana Page Views (by path) | Supports #2 Auth Friction | Shows login friction and dashboard consumption | 4,569 total |
| 752533597 | total active time where URL domain contains github.com | single_number | GitHub Active Time | Baseline context | Total time investment in GitHub | 57,868 min (~964h) |
| 805213806 | total active time where URL domain contains fullstory.atlassian.net | single_number | Jira Active Time | Baseline context | Total time investment in Jira | 58,773 min (~980h) |
| 1469839871 | total active time where URL domain contains app.circleci.com | single_number | CircleCI Active Time | Primary signal for #1 Pipeline Polling | Measures total CI tool time investment | 9,611 min (~160h) |
| 329101188 | total active time where URL domain contains fullstory.grafana.net | single_number | Grafana Active Time | Baseline context | Total observability tool time | 12,572 min (~210h) |
| 790720192 | unique users where URL domain contains github.com | single_number | GitHub Unique Users | Baseline context | Team size using GitHub | 205 |
| 541833510 | unique users where URL domain contains fullstory.atlassian.net | single_number | Jira Unique Users | Baseline context | Team size using Jira | 306 |
| 696665180 | page views where URL path contains login AND URL domain contains app.circleci.com | single_number | CircleCI Login Views | Primary signal for #2 Auth Friction | Directly measures CircleCI auth interruptions | 418 |
| 1258367953 | page views where URL path contains login AND URL domain contains fullstory.grafana.net | single_number | Grafana Login Views | Primary signal for #2 Auth Friction | Directly measures Grafana auth interruptions | 309 |
| 180057838 | page views where URL path contains pulls AND URL domain contains github.com | single_number | GitHub PR Page Views | Primary signal for #5 PR Navigation | Measures PR review navigation volume | 3,418 |
| 1694880875 | page views where URL path contains backlog AND URL domain contains fullstory.atlassian.net | single_number | Jira Backlog Views | Primary signal for #3 Board Polling | Directly measures backlog polling/grooming frequency | 2,312 |
| 1811475553 | count of Jira Visit events | single_number | Jira Visit Events | Baseline context | Total Jira interaction volume across all users | 63,568 |

## Suggested Dashboard Names

- `[WFA-FSWorkforce-2026-05] CircleCI PV` — Pipeline polling primary signal
- `[WFA-FSWorkforce-2026-05] CircleCI Time` — Pipeline time investment
- `[WFA-FSWorkforce-2026-05] CircleCI Login` — Auth friction (CircleCI)
- `[WFA-FSWorkforce-2026-05] Grafana Login` — Auth friction (Grafana)
- `[WFA-FSWorkforce-2026-05] GitHub Auth` — Auth friction (GitHub SAML)
- `[WFA-FSWorkforce-2026-05] Jira Backlog` — Board/backlog polling
- `[WFA-FSWorkforce-2026-05] GitHub PRs` — PR review navigation
- `[WFA-FSWorkforce-2026-05] Jira Visits` — Overall Jira volume
- `[WFA-FSWorkforce-2026-05] GitHub Time` — GitHub time baseline
- `[WFA-FSWorkforce-2026-05] Jira Time` — Jira time baseline
