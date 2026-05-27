---
name: workforce-automation-analysis
description: >-
  Run a FullStory Workforce automation discovery analysis for a customer org.
  Identifies manual workflows, quantifies time waste, reviews sessions for
  behavioral evidence, and produces a polished HTML deliverable with ranked
  automation candidates and implementation roadmaps. Use when the user asks
  to analyze workforce data, find automation opportunities, run a workforce
  automation engagement, or build an automation blueprint.
---

# Workforce Automation Analysis

A structured playbook for discovering and quantifying process automation
opportunities using FullStory Workforce data. Produces a single self-contained
HTML deliverable with quantitative evidence, session replay proof points, and
implementation roadmaps.

## Overview

The analysis runs in 7 phases per team/business unit:

| Phase | Purpose | Primary Tools |
|-------|---------|---------------|
| 0 | Orientation — discover tools, users, scope teams | `build_metric`, `discover_org_context` |
| 0.5 | Metric Persistence — save/reference metrics for benchmarking | `get_metric` (if referencing) |
| 1 | Quantitative Discovery — volumes, signals | `build_metric`, `compute_metric`, `discover_org_context` |
| 2 | Session Deep Dives — behavioral evidence | `get_sessions`, `get_session_events` (via subagents) |
| 3 | Synthesis — score and rank candidates | Analysis of Phase 1 + 2 outputs |
| 4 | Deliverable — generate HTML report | Write using report-template.html |
| 5 | Post-Implementation Measurement — before/after proof | `get_metric`, `compute_metric`, `get_sessions` |

---

## Phase 0: Orientation

Goal: Build a data-driven inventory of the org's tool stack and agree on scope.

### 0.1 Discover the Tool Inventory

Do NOT hardcode a list of tools. Query the org's actual captured data:

1. `build_metric`: query="page views grouped by top-level URL domain",
   output_type="top_n", time_range="last_30_days"
2. `compute_metric` with the returned metric_id

This returns every domain with captured activity, ranked by volume. No
dependency on page definitions or org configuration maturity.

### 0.2 Discover User Context

Pull the **full user object schema** from the org — do NOT search for a
hardcoded list of property names. User properties vary wildly between orgs.

```
discover_org_context(queries=["user properties", "user schema",
  "user custom variables"])
```

Evaluate the returned schema and identify any properties that represent:
- Team / department / group membership
- Role / title / function
- Region / office / location
- Tenure / start date / seniority
- Any other property that helps segment users into meaningful work groups

**Guidelines**:
- Pull the full schema and assess — do NOT search for specific names
- Properties with recognizable, descriptive names and human-readable
  values are useful (e.g., `department: "Support"`, `role: "Agent"`)
- Ignore opaque internal data: hashed IDs, system-generated codes,
  deeply nested objects, or values that aren't interpretable without
  org-specific knowledge
- If team/department/role-equivalent properties exist → use them to
  ground team groupings in Phase 0.4
- If no meaningful user properties exist → fall back to tool-cluster
  inference and note the limitation in the deliverable

### 0.3 Enrich with Active Time

For each top domain (ignore auth/SSO domains, CDNs, etc.):

1. `build_metric`: query="total active time where URL domain contains
   {domain}", output_type="single_number"
2. `compute_metric` to get hours spent per tool

### 0.4 Present and Scope

Present the ranked tool inventory to the user with page views + active
time. If Phase 0.2 revealed user properties with team/department/role
data, show team groupings grounded in actual user data:

> "34 users with department=Support spend 80% of time in Zendesk"

If no meaningful user properties exist, recommend team groupings based
on observed tool clusters as a fallback:

| Tool Cluster | Suggested Team |
|-------------|---------------|
| Zendesk, Intercom, Freshdesk, ServiceNow | Customer Support |
| SFDC + Outreach/Salesloft + Gong/Chorus + ZoomInfo | Sales Operations |
| SFDC + Looker/Tableau + Gong + CPQ | RevOps / Deal Desk |
| Jira, Linear, Confluence, GitHub | Engineering Ops |
| HubSpot, Marketo, LinkedIn, Drift | Marketing Ops |
| Workday, BambooHR, Greenhouse, Lever | People Ops / HR |

These are suggestions — the user decides which teams to analyze. Recommend
2–3 teams for a comprehensive engagement. The highest-volume tools should
get priority since that's where the most employee time goes.

**Record the user-confirmed team names exactly as stated.** These become
the tab labels in the deliverable. Do NOT rename or normalize them.

### 0.4.1 Confirm Parameters

- Teams to analyze (user-confirmed names + which tools map to each)
- Time range (default: last 30 days)
- Number of session deep dives per team (default: 3)

---

## Phase 0.5: Metric Persistence

Goal: Decide whether to track metrics for later dashboarding, benchmarking,
and before/after measurement.

### 0.5.1 Ask the User

Use the `AskQuestion` tool:

1. "Would you like me to create a metric ledger for this analysis? This
   tracks all metric definitions with descriptions so they can be re-used
   for FullStory dashboards and before/after benchmarking."
2. "Is there an existing analysis to reference? If so, provide the prefix
   (e.g., WFA-Acme-2026-05)."

### 0.5.2 If Saving (New Analysis)

Establish a prefix: `WFA-{OrgShortName}-{YYYY-MM}`

Create a ledger file at:
`~/.cursor/skills/workforce-automation-analysis/ledgers/{prefix}.md`

Initialize with headers:

```markdown
# Metric Ledger: {prefix}

Org: {org_id}
Date Range: {start} – {end}
Teams: {team1}, {team2}, ...
Created: {timestamp}

| Metric ID | Query | Output Type | Signal Name | Blueprint Role | Significance | Baseline Value |
|-----------|-------|-------------|-------------|---------------|--------------|----------------|
```

After each `build_metric` → `compute_metric` sequence in Phase 1, append
a row to the ledger:

- **Signal Name**: human-readable label (e.g., "Copied Comment Log Entry")
- **Blueprint Role**: which automation opportunity this supports (set to
  "TBD" during Phase 1, backfilled during Phase 3)
- **Significance**: why this metric matters for the analysis (backfilled
  during Phase 3)
- **Baseline Value**: the computed value at time of analysis

### 0.5.3 If Referencing (Existing Analysis)

Use `get_metric` with `regex` matching the prefix to find previously saved
metrics. Where the same query already exists, skip redundant `build_metric`
calls and use the existing metric ID for `compute_metric`.

Note: Metrics created by `build_metric` are unnamed in FullStory. The
ledger file is the source of truth for mapping metric IDs to their purpose.
The Measurement & Proof tab in the deliverable (Phase 4) provides clickable
FullStory links so the user can manually name and save metrics for dashboard
use.

---

## Phase 1: Quantitative Discovery

Run for each team. All queries use URL-based filtering (not page definitions)
to ensure completeness regardless of org configuration maturity.

See [query-playbook.md](query-playbook.md) for detailed query sequences
by team type.

### 1.1 Volume & Time (per tool in the team's stack)

Build and compute these metrics for each tool domain:
- `top_n`: "page views grouped by URL path where domain contains {domain}"
- `single_number`: "unique users where URL domain contains {domain}"
- `single_number`: "total active time where URL domain contains {domain}"

### 1.2 Interaction Signals

Use `discover_org_context` with queries derived from the team's tools
(NOT a hardcoded list — use the actual tool names from Phase 0):

```
discover_org_context(queries=["zendesk", "ticket", "submit", "copy", ...])
```

Look for defined events and named elements related to:
- Copy/paste (manual data transfer signal)
- Form submissions (data entry volume)
- Field changes (manual field population)
- Search/lookup (research pattern)
- Specific button clicks (workflow triggers)

For each relevant signal found, build a `single_number` metric to count
occurrences over the time range.

### 1.3 Behavioral Patterns

Analyze the URL path breakdown from 1.1 to identify:
- **Data consumption**: report/dashboard paths (high view counts = manual
  data lookup that could be surfaced inline)
- **Queue/list views**: ticket queues, task lists (high counts = manual
  polling/triage)
- **Record-level views**: individual records (high counts relative to
  unique records = repetitive lookup)
- **Search frequency**: search paths (high counts = context loss or
  poor navigation)
- **Low-usage areas**: KB, documentation, help (low counts = underused
  resources that could reduce other work)

### 1.4 Document Findings

Record structured findings per tool:
```
Tool: {name}
Domain: {domain}
Page Views: {count}
Active Time: {hours}h
Users: {count}
Top Paths: {path → count list}
Notable Signals: {defined events and their counts}
Anomalies: {anything unexpected}
```

---

## Phase 2: Session Deep Dives

Goal: Observe actual employee behavior to identify manual workflow loops,
friction points, and automation opportunities that quantitative data alone
cannot reveal.

### 2.1 Session Selection

For each team, select 3 sessions using `get_sessions` with a metric_id
targeting the team's primary tool. Aim for:
- **High page count** (sustained work sessions, not drive-by visits)
- **Multiple tool domains** if possible (cross-tool switching)
- **Presence of friction signals** (rage clicks, dead clicks, mouse thrash)

If the first batch doesn't yield good candidates, try different metrics
(e.g., copy-paste events, specific workflow triggers).

### 2.2 Session Analysis

For each session, launch a `session-context` subagent with `get_session_events`
and this task:

> Extract a timestamped behavioral transcript of this session. For each
> distinct workflow pattern, document:
> 1. What the user did (step-by-step actions)
> 2. How long it took (timestamps)
> 3. How many times it repeated
> 4. What friction signals appeared (rage clicks, long pauses, dead clicks)
> 5. What cross-tool switching occurred (long gaps suggesting alt-tab)
> 6. What a machine could do instead
>
> Focus on: repetitive loops (same sequence 3+ times), copy-paste chains,
> manual data entry, navigation inefficiency (3+ page loads for one task),
> queue polling (manual refresh), and duplicate lookups (same record opened
> twice).

### 2.3 Extract Evidence Clips

From each session analysis, identify the 2–3 most compelling moments that
illustrate manual work a machine should handle. Record:
- Session URL: `https://app.fullstory.com/ui/{org-id}/session/{device-id}:{session-id}`
- Timestamp (minutes:seconds from session start)
- Description of what's happening
- Which automation candidate it supports

These become the "Watch It Happen" links in the deliverable.

---

## Phase 3: Synthesis & Scoring

See [scoring-rubric.md](scoring-rubric.md) for the detailed methodology.

### 3.1 Identify Automation Candidates

For each manual workflow observed across Phases 1 and 2, create a candidate:

| Field | Source |
|-------|--------|
| Name | Descriptive workflow name |
| Signal from Data | Which Phase 1 metrics support this |
| Session Evidence | Specific timestamps from Phase 2 |
| Monthly Event Volume | From Phase 1 metric counts |
| Time per Event | Conservative estimate from session observation |
| Eliminability % | What percentage automation could handle |
| Monthly Hours Saved | volume × time_per_event × eliminability ÷ 60 |

Target 5–6 candidates per team.

### 3.2 Classify Automation Level

- **Fully Automated**: Zero human intervention. Trigger → execute → done.
  Examples: auto-enrichment on record creation, bidirectional CRM sync,
  auto-populate fields from parsed input, SSO/auth fixes.

- **AI-Assisted**: AI does heavy lifting, human reviews/approves.
  Examples: AI-drafted responses, call summary generation, smart task
  prioritization, intelligent search/surfacing.

- **Process Automated**: Smart routing, data consolidation, fewer steps.
  Examples: unified dashboards replacing cross-tool lookup, batch actions
  replacing one-by-one processing, auto-classification/routing.

### 3.3 Rank Candidates

Sort all candidates across all teams by:
1. Monthly hours saved (primary — descending)
2. Implementation complexity (secondary — ascending)
3. Automation level (tertiary — Fully Automated first)

### 3.4 Backfill Metric Ledger

If metric persistence is enabled (Phase 0.5), go back through the ledger
and fill in the "Blueprint Role" and "Significance" columns now that
automation candidates are identified. Each metric should be tagged with:

- Which opportunity it's a **primary signal** for (the metric that directly
  measures the manual work the automation will eliminate)
- Which opportunity it **supports** (context metrics like total volume,
  active time breakdowns, etc.)
- **Baseline context** for metrics that don't map to a specific opportunity
  but ground the overall analysis

Also add a suggested dashboard name for each metric:
`[{prefix}] {Signal Name}` — e.g., `[WFA-Acme-2026-05] Copied Comment Log`

### 3.5 Build Implementation Roadmap (per team)

Group candidates into phases:
- **Phase 1 — Quick Wins**: Fully Automated items with low implementation
  effort (config changes, existing integrations)
- **Phase 2 — High Impact**: AI-Assisted items requiring new tooling or
  integration work
- **Phase 3 — Foundation**: Process Automated items requiring workflow
  redesign or cross-tool architecture

---

## Phase 4: Deliverable

Generate a single self-contained HTML file using the template structure
in [report-template.html](report-template.html).

### Structure

The deliverable uses tab-based navigation. **Tabs are NOT hardcoded** —
they map to the teams the user confirmed in Phase 0.5.

**Tab order**:
1. Overview (always present, always first)
2. One tab per confirmed team, using the exact name the user provided
3. Measurement & Proof (always present, always last)

**Team accent colors** assigned by tab order:
- Team 1: purple (#6c5ce7, #a29bfe)
- Team 2: blue (#3b82f6, #60a5fa)
- Team 3: green (#10b981, #34d399)
- Team 4: amber (#f59e0b, #fbbf24)
- Team 5+: cycle from the beginning

**Overview Tab**:
- Hero with aggregate stats (total hours saved, FTE equivalent, workflow
  count, session count)
- Cards linking to each team blueprint tab (one card per confirmed team)
- Summary bar by automation level (Fully Automated / AI-Assisted /
  Process Automated)
- Ranked table of ALL candidates across all teams with: rank, team,
  workflow name, automation level, hours saved, primary tools, and
  anchor link to the candidate card
- "Key Moments" section: 8–10 curated session replay links with
  timestamps and descriptions

**Per-Team Blueprint Tabs** (one per user-confirmed team):
- Tab label = the exact team name the user confirmed in Phase 0.5
- Executive callout with key numbers
- Tool volume breakdown (page views, active time, users per tool)
- URL path analysis with bar charts showing where time goes
- Behavioral signal analysis (copy-paste counts, field changes, etc.)
- Automation candidate cards, each containing:
  - Signal from data + estimated savings
  - Current state → future state description
  - Session evidence block
  - "Watch It Happen" evidence clips with timestamped FullStory links
- Automation impact summary table
- Implementation roadmap (phased)
- Session replay links section

**Measurement & Proof Tab** (always present as the last tab):
- **Baseline Metrics Table**: All key signal metrics with:
  - Metric ID and FullStory link
    (`https://app.fullstory.com/ui/{org-id}/metrics/create/{metric_id}`)
  - Signal name and description
  - Current baseline value (from Phase 1)
  - Suggested dashboard name (`[{prefix}] {Signal Name}`)
  - Which opportunity it maps to (primary signal vs supporting)
- **Per-Opportunity Success Criteria**: Table with columns for baseline,
  target reduction %, and checkpoints at 30/60/90 days post-implementation
- **Before/After Template**: Pre-filled baseline column with empty
  post-implementation columns for 30/60/90 day results
- **Dashboard Setup Guide**: Instructions for clicking each metric link
  in FullStory → naming it with the suggested name → saving → adding to
  a dashboard
- **Re-Measurement Instructions**: How to run Phase 5 (post-implementation
  measurement) or manually re-compute metrics with shifted date ranges
- If user properties were not available in Phase 0.2, include a note:
  "Team groupings in this analysis are inferred from tool usage patterns.
  Configuring user properties (department, role, team) in FullStory will
  enable more precise team segmentation in future analyses."

### Design Requirements

- Single file, zero external dependencies
- All CSS inline in one `<style>` block, all JS in one `<script>` block
- Tab switching via JavaScript `switchTab(tabId, anchor)` function
- Purple accent (#6c5ce7) for FullStory branding
- SVG charts for trends and distributions (no chart libraries)
- Responsive layout with print support
- All FullStory session links use `target="_blank"`
- Evidence clips styled with purple background, bold timestamps

### Hosting

The file can be:
- Opened directly in a browser (double-click)
- Uploaded to Google Apps Script as a single `index.html` with a minimal
  `doGet()` function
- Hosted on any static file server
- Attached to an email or Slack message

---

## Phase 5: Post-Implementation Measurement

Goal: Re-run the analysis metrics after automation implementation to
produce a before/after proof report demonstrating impact.

### When to Use

This phase is triggered when the user returns after implementing one or
more automation opportunities and says "run the proof report," "measure
the results," "show me the before/after," or similar.

### 5.1 Gather Inputs

Confirm with the user:
1. **Analysis prefix** — to load the metric ledger
   (e.g., `WFA-Acme-2026-05`)
2. **Implementation date** — when the automation(s) went live
3. **Which opportunities were implemented** — to know which metrics to
   focus on
4. **Agent hourly cost** (optional) — for ROI dollar calculation

### 5.2 Load the Metric Ledger

Read the ledger file at:
`~/.cursor/skills/workforce-automation-analysis/ledgers/{prefix}.md`

If no ledger exists (analysis was run before metric persistence was
added), reconstruct from the original analysis by searching the
conversation history or asking the user which metric IDs to track.

### 5.3 Compute Before/After

For each metric in the ledger tagged with an implemented opportunity:

1. `compute_metric` with `start_date`/`end_date` for the **pre-
   implementation** window (same date range as the original analysis)
   → locks the baseline value
2. `compute_metric` with `start_date` = implementation date,
   `end_date` = today → measures post-implementation state
3. Calculate:
   - Absolute delta (baseline − current)
   - Percentage change
   - Whether it meets the target from the success criteria

### 5.4 Qualitative Validation

Pull 2–3 post-implementation sessions using the same session selection
criteria as Phase 2 (high page count, sustained work sessions in the
team's primary tool). Analyze via `session-context` subagents to
observe how the workflow has changed.

### 5.5 Generate Proof Report

Produce a **proof report HTML** (same styling as the Phase 4 deliverable)
with:

- **Hero**: headline delta ("X hours/month reclaimed" or similar)
- **Before/After metrics table**: per-metric comparison with green
  (improved) / red (regressed) delta indicators
- **Per-opportunity breakdown**: baseline → current → delta → target →
  on-track status
- **Session comparison**: "Before" session link (from original analysis)
  alongside "After" session link (from Phase 5.4)
- **ROI calculation**: hours saved × agent hourly cost (if provided)
- **Recommendations**: which remaining opportunities to tackle next,
  any unexpected findings

Save as `proof-report-{prefix}-{YYYY-MM}.html` alongside the original
deliverable.
