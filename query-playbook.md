# Query Playbook

Detailed MCP query sequences organized by team type. Each section lists
the exact queries to run, in order, with the tool and parameters.

All queries use URL-based filtering to ensure completeness regardless of
org configuration maturity. Page definitions and defined events are used
for enrichment when available, never as the primary data source.

**Metric Ledger**: If metric persistence is enabled (Phase 0.5), after
each `build_metric` → `compute_metric` pair, append a row to the ledger
file with the metric_id, query, output_type, signal name, and computed
value. Leave "Blueprint Role" and "Significance" as "TBD" — these are
backfilled during Phase 3 synthesis.

---

## Universal Queries (run for every engagement)

### Tool Inventory Discovery

```
build_metric:
  query: "page views grouped by top-level URL domain"
  output_type: top_n
  time_range: last_30_days
→ compute_metric(metric_id)
→ ledger: Signal="Tool Inventory (by domain)", Role="Baseline context"
```

Result: ranked list of every domain with captured activity. Use this to
identify the org's tool stack.

### User Context Discovery

Pull the full user object schema — do NOT search for hardcoded names:

```
discover_org_context:
  queries: ["user properties", "user schema", "user custom variables"]
```

Evaluate the returned schema. Identify any properties that could
represent team, department, role, title, region, tenure, or similar.
Properties with recognizable names and human-readable values are useful.
Ignore opaque internal data (hashed IDs, system codes, nested objects).

If team/department/role-equivalent properties exist, use them to build
user segments:

```
build_segment:
  query: "users where {property_name} equals {team_value}"
  name: "{Team Name} Users"
→ Use these segments to scope Phase 1 metrics per team
```

### Per-Tool Volume Baseline

For each relevant domain discovered above:

```
build_metric:
  query: "page views grouped by URL path where URL domain contains {domain}"
  output_type: top_n
→ compute_metric(metric_id)
→ ledger: Signal="{Domain} Page Views (by path)", Role="Baseline context"

build_metric:
  query: "total active time where URL domain contains {domain}"
  output_type: single_number
→ compute_metric(metric_id)
→ ledger: Signal="{Domain} Active Time", Role="Baseline context"

build_metric:
  query: "unique users where URL domain contains {domain}"
  output_type: single_number
→ compute_metric(metric_id)
→ ledger: Signal="{Domain} Unique Users", Role="Baseline context"
```

### Org Context Discovery

After identifying the tools, search for defined events and named elements:

```
discover_org_context:
  queries: ["{tool1_name}", "{tool2_name}", "copy", "paste", "submit",
            "click", "field", "search", "ticket", "deal", "task"]
```

Adapt the queries based on what tools were discovered. The goal is to find
org-specific instrumentation that provides richer signals than raw page views.

---

## Customer Support Team

Primary tools: Zendesk, Intercom, Freshdesk, ServiceNow, or similar.

**Note**: Team names in this playbook are examples. Use the actual team
name confirmed by the user in Phase 0.5.

### Volume Queries

```
# Ticket volume (look for defined events first)
discover_org_context(queries=["{support_tool}", "ticket", "submit", "solved"])

# If defined events exist for ticket actions:
build_metric:
  query: "count of {defined_event_name} events"
  output_type: single_number
→ compute_metric(metric_id)
→ ledger: Signal="{event_name}", Role="TBD"

# Ticket dispositions (if events exist for solved/pending/on-hold)
build_metric:
  query: "count of {solved_event} events"
  output_type: single_number
→ ledger: Signal="{disposition} count", Role="TBD"
# Repeat for each disposition type

# Agent reply volume
build_metric:
  query: "count of {reply_event} events"
  output_type: single_number
→ ledger: Signal="Agent reply count", Role="TBD"
```

### Interaction Signal Queries

```
# Copy-paste activity (critical for support — indicates manual response assembly)
discover_org_context(queries=["copy", "paste", "clipboard"])

# For each copy/paste defined event found:
build_metric:
  query: "count of {copy_event} events"
  output_type: single_number
→ ledger: Signal="{copy_event}", Role="TBD"

# If granular copy events exist (e.g., "Copied Comment Log", "Copied Previous Message"):
build_metric:
  query: "count of {specific_copy_event} events"
  output_type: single_number
→ ledger: Signal="{specific_copy_event}", Role="TBD"
# Run for each variant to understand WHERE agents are copying FROM

# Manual data entry signals
discover_org_context(queries=["org id", "input", "field", "category", "follow-up"])

# For each data entry event found:
build_metric:
  query: "count of {field_event} events"
  output_type: single_number
→ ledger: Signal="{field_event}", Role="TBD"

# Knowledge base usage (or lack thereof)
build_metric:
  query: "page views where URL path contains help OR knowledge OR guide
          AND URL domain contains {support_tool_domain}"
  output_type: single_number
→ ledger: Signal="KB Page Views", Role="TBD"
```

### Session Selection

```
# Find sessions with high activity in the support tool
build_metric:
  query: "page views where URL domain contains {support_tool_domain}"
  output_type: single_number
→ get_sessions(metric_id, limit=10)

# Select 3 sessions with highest page counts
# Prefer sessions showing sustained work (not just a few page views)
```

### What to Look For in Sessions

- Ticket cycle pattern: queue → open → read → compose → set fields → submit
- Copy-paste chains: copying from old tickets/messages into new replies
- Field population rituals: repetitive dropdown/field setting on every ticket
- Cross-ticket reference: switching between tickets to find past resolutions
- Queue refresh behavior: manual view refreshing while waiting for tickets
- Knowledge base avoidance: agents using old tickets instead of KB articles

---

## Sales Operations Team

Primary tools: SFDC, Outreach/Salesloft, Gong/Chorus, ZoomInfo, Clay, LinkedIn.

### Volume Queries

```
# SFDC object breakdown
build_metric:
  query: "page views grouped by URL path where URL domain contains salesforce"
  output_type: top_n
→ compute_metric(metric_id)

# Outreach breakdown (check for both web and sidebar domains)
build_metric:
  query: "page views grouped by URL path where URL domain contains outreach"
  output_type: top_n
→ compute_metric(metric_id)

# Gong activity
build_metric:
  query: "page views grouped by URL path where URL domain contains gong"
  output_type: top_n
→ compute_metric(metric_id)

# Enrichment tool usage
build_metric:
  query: "page views where URL domain contains zoominfo"
  output_type: single_number

build_metric:
  query: "page views where URL domain contains clay"
  output_type: single_number
```

### Interaction Signal Queries

```
# Prospect creation (Outreach sidebar)
discover_org_context(queries=["prospect", "outreach", "sequence", "task"])

# Call intelligence interactions
discover_org_context(queries=["gong", "call", "recording", "account"])

# SFDC enrichment triggers
discover_org_context(queries=["enrichment", "clay", "launch"])

# For each signal found, build count metrics as above
```

### Session Selection

```
# Find sessions with cross-tool switching (SFDC + Outreach)
build_metric:
  query: "page views where URL domain contains salesforce"
  output_type: single_number
→ get_sessions(metric_id, limit=10)

# Also look for Outreach-heavy sessions
build_metric:
  query: "page views where URL domain contains outreach"
  output_type: single_number
→ get_sessions(metric_id, limit=10)

# Select 3 sessions showing different workflow patterns
```

### What to Look For in Sessions

- Report-to-record drill-down loops: navigating from reports to accounts to
  contacts repeatedly (prospecting workflow)
- Prospect creation grind: manually creating prospects one by one in sidebar
- Sequence task execution: processing tasks one at a time with context gaps
- Cross-tool context switching: long pauses (30s+) indicating alt-tab to
  another app for information
- Duplicate lookups: same record searched/opened more than once (context loss)
- Manual call logging: typing notes that originated from Gong recordings
- Data table rage clicks: frustration with list/table UI interactions

---

## RevOps / Deal Desk Team

Primary tools: SFDC (reports, dashboards, CPQ), Looker/Tableau, Gong.

### Volume Queries

```
# SFDC reports and dashboards specifically
build_metric:
  query: "page views where URL path contains report OR dashboard
          AND URL domain contains salesforce"
  output_type: single_number

# Looker/Tableau activity
build_metric:
  query: "page views grouped by URL path where URL domain contains
          {analytics_tool_domain}"
  output_type: top_n
→ compute_metric(metric_id)

# Deal desk / quoting activity
discover_org_context(queries=["deal desk", "quote", "CPQ", "approval"])

# For each deal desk event found:
build_metric:
  query: "count of {deal_desk_event} events"
  output_type: single_number
```

### Interaction Signal Queries

```
# Report consumption patterns
build_metric:
  query: "page views where URL path contains /report
          AND URL domain contains salesforce"
  output_type: single_number

build_metric:
  query: "page views where URL path contains /dashboard
          AND URL domain contains salesforce"
  output_type: single_number

# Gong-to-SFDC transfer signals
build_metric:
  query: "page views where URL domain contains gong"
  output_type: single_number

# Login/auth friction (especially analytics tools)
build_metric:
  query: "page views where URL path contains login OR auth OR oauth
          AND URL domain contains {analytics_tool_domain}"
  output_type: single_number

# Flow Builder / automation work (indicates team is already investing)
build_metric:
  query: "page views where URL path contains flow
          AND URL domain contains salesforce"
  output_type: single_number
```

### Session Selection

```
# Find deal-heavy sessions in SFDC
build_metric:
  query: "page views where URL path contains opportunity OR quote
          AND URL domain contains salesforce"
  output_type: single_number
→ get_sessions(metric_id, limit=10)
```

### What to Look For in Sessions

- Dashboard-as-checklist: opening a report, then batch-opening records from it
  (report is navigation, not analytics)
- Deal desk form friction: time spent filling out request forms, abandoned forms
- Manual back-fill: editing Opportunity fields with data from a Quote (post-
  quote save-edit cycles)
- Pipeline hygiene rituals: identical next-step updates on multiple opps
  (e.g., typing the same date prefix pattern)
- Cross-tool data consumption: switching between SFDC reports, Looker, and Gong
  to assemble deal context
- Auth friction: re-authentication interrupting workflows (especially Looker)

---

## Engineering Ops Team

Primary tools: Jira, Linear, GitHub, Confluence, CI/CD dashboards.

### Volume Queries

```
build_metric:
  query: "page views grouped by URL path where URL domain contains
          {project_tool_domain}"
  output_type: top_n

build_metric:
  query: "page views grouped by URL path where URL domain contains github"
  output_type: top_n
```

### What to Look For in Sessions

- Ticket grooming overhead: time spent updating status, priority, sprint fields
- Context switching: bouncing between Jira/Linear and GitHub for the same work item
- Standup/reporting rituals: manually compiling status from multiple sources
- PR review workflows: navigation patterns during code review
- Documentation hunting: searching Confluence/wiki for information

---

## Marketing Ops Team

Primary tools: HubSpot, Marketo, LinkedIn Ads, Drift, analytics dashboards.

### Volume Queries

```
build_metric:
  query: "page views grouped by URL path where URL domain contains
          {marketing_tool_domain}"
  output_type: top_n
```

### What to Look For in Sessions

- Campaign creation overhead: multi-step form filling for new campaigns
- Lead routing: manual lead assignment/qualification workflows
- Reporting assembly: pulling data from multiple tools for performance reports
- List management: manual list building, deduplication, segmentation
- Content publishing workflows: multi-step approval/publishing processes

---

## Adapting to Unknown Tools

If the org uses tools not listed above:

1. The URL path breakdown (`top_n` by path) reveals the tool's internal
   structure — what objects/views exist
2. `discover_org_context` with the tool name reveals any custom instrumentation
3. Session deep dives reveal the actual workflows regardless of whether you
   recognize the tool

The methodology works for ANY web application captured by FullStory Workforce.
The queries adapt; the analysis framework stays the same.

---

## Ledger Backfill (Phase 3)

During Phase 3 synthesis, revisit every row in the metric ledger and
update the "Blueprint Role" and "Significance" columns:

```markdown
| Metric ID | Query | Output Type | Signal Name | Blueprint Role | Significance | Baseline Value |
|-----------|-------|-------------|-------------|----------------|--------------|----------------|
| m-abc123  | count of Copied Comment Log events | single_number | Copied Comment Log | Primary signal for #1 Response Assembly | Directly measures copy-paste volume that AI drafting eliminates | 2,847 |
| m-def456  | total active time where domain contains zendesk | single_number | Zendesk Active Time | Baseline context | Frames total support tool time investment | 412h |
```

For each metric, add a suggested FullStory dashboard name:
`[{prefix}] {Signal Name}`

---

## Post-Implementation Re-Measurement (Phase 5)

When the user returns for proof measurement, re-compute each ledger
metric with a shifted date range:

```
# For each metric_id in the ledger:
compute_metric:
  metric_id: {metric_id}
  start_date: {implementation_date}
  end_date: {today}
→ Compare to "Baseline Value" column in the ledger
→ Calculate absolute delta, percentage change, on-track status
```

If the original metric_id is no longer valid (expired, deleted), rebuild
the same query with `build_metric` and compute over the post-
implementation date range.
