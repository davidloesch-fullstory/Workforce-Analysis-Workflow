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

Goal: the COMPLETE host list (see SKILL.md Phase 0.1 for the full procedure).
Force the host dimension, beat the 50-row cap with a raw high-limit definition,
then verify coverage. Keep ALL hosts (including auth/SSO, CDN, internal).

```
build_metric:
  query: "page views grouped by URL host"
  output_type: top_n
  start_date: {analysis_start}   # absolute window from Phase 0.0
  end_date:   {analysis_end}     # never use time_range for persisted metrics
→ verify definition: dimension.builtInProperty == "DIMENSION_URL_HOST"
   (if it came back DIMENSION_URL, update_metric refinement:
    "group by URL host instead of the full URL")
→ compute_metric via metric_definition escape hatch with limit raised to 200
   (the default 50 is a UI/builder cap; the server honors higher limits)
→ coverage check: sum(returned rows) ÷ result.total × 100
   (100% = complete; if rows == limit and < 100%, raise limit or paginate)
→ ledger: Signal="Tool Inventory (by host)", Role="Baseline context",
          note coverage % and that an unnamed metric was created
```

Result: the full list of hosts with captured activity. Collapse to registrable
domain for the human-facing inventory (keep raw hosts too). Note: the saved
metric's FullStory link renders only the top 50 rows (UI cap) — add a
tooltip/footnote when linking it.

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

Enumerate the property's distinct **values** (read from the schema sample +
confirm with the user — there is no "list all values" tool). Lock a
`{team → property=value}` mapping and a per-team scoping mode (people-scoped vs
tool-cluster).

For each **people-scoped** team, build a membership segment pinned to the window:

```
build_segment:
  query: "users where {property_name} = {team_value}"
  name:  "{prefix} {Team Name} Members"
  start_date: {analysis_start}   # pin to the Phase 0.0 window
  end_date:   {analysis_end}
→ ledger Segments table: Segment ID, Team, "{property}={value}", Window, "people-scoped"
```

### Per-Tool Volume Baseline

For each relevant domain. People-scoped teams attach the team segment so the
numbers reflect the team's members; tool-cluster teams leave metrics URL-scoped.

```
# Build the metric (WHAT), then for people-scoped teams attach segment (WHO):
build_metric:
  query: "page views grouped by URL path where URL domain contains {domain}"
  output_type: top_n
  start_date: {analysis_start}    # always pin the window
  end_date:   {analysis_end}
→ (people-scoped) update_metric(metric_id, segment_id={team_segment})  # new metric_id
→ compute_metric(metric_id)
→ ledger: Signal="{Domain} Page Views (by path)", Segment ID={team_segment|blank}, Role="Baseline context"

# focused (engaged) time — NL maps "focused" → active, so clone-and-swap:
build_metric: "total Page active time (new) where URL domain contains {domain}" (single_number)
→ (people-scoped) update_metric(..., segment_id)
→ clone returned metric_definition, set property.builtInProperty=28, compute_metric(metric_definition)
build_metric: "unique users where URL domain contains {domain}" (single_number)
→ (people-scoped) update_metric(..., segment_id) → compute_metric
```

Use **engaged time = `Page focused time (new)`** as the primary measure — it
counts focused/foreground attention incl. reading and reviewing, which `Page
active time (new)` (interaction only) undercounts. **Measure focused time ONLY
via the clone-and-swap escape hatch** (`builtInProperty: 28`); the NL builder
silently maps "focused" → active. Enums (event node `visitedPage`, agg
`NUMERIC_AGGREGATION_METHOD_SUM`): focused = `28` (numeric), active =
`"NUMERIC_PROPERTY_PAGEVIEW_ACTIVE_DURATION"`, visible =
`"NUMERIC_PROPERTY_PAGEVIEW_TOTAL_DURATION"`. Verify visible ≥ focused ≥ active.

**Cross-tool team view (people-scoped, headline):** take the `DIMENSION_URL_HOST`
breakdown from Tool Inventory Discovery, swap its aggregated property to focused
(`builtInProperty: 28`) via the escape hatch, attach the team segment, and
compute → the team's full focused-time distribution across their entire stack.

**Primary tool only — adoption + work-type ratio (people-scoped):**
```
# active time on the primary tool via NL (maps correctly), unscoped + team-scoped:
build_metric: "total Page active time (new) where URL domain contains {primary_domain}" (single_number)
# focused = SAME definition with property.builtInProperty=28 (clone-and-swap):
→ compute_metric(metric_definition with builtInProperty=28)   # engaged hours
# adoption: team-scoped focused ÷ tool-only focused = team share ("Support = 80% of all Zendesk")
# work-type ratio: active ÷ focused → high = input-heavy (auto-populate), low = read-heavy (AI-assist)
```

### Friction & Error Discovery (always-on)

Surface manual-workaround signals via top frustration/error groups:

```
# people-scoped team:
discover_groups(segment_id={team_segment}, limit=5,
                start_time={analysis_start}, end_time={analysis_end})
# primary tool (any team):
discover_groups(domain={primary_tool_host}, limit=5,
                start_time={analysis_start}, end_time={analysis_end})
```

Capture each group + its `metric_url` as candidate evidence for Phase 3.

### Interaction Signals (tiered — see SKILL.md Phase 1.2)

Quantify manual-work signals (copy/paste, submissions, field changes, search,
workflow clicks). **Names are hints, raw behavior is truth** — never depend on
clean instrumentation. Work down the tiers; for people-scoped teams attach the
team segment to every metric.

**Tier A — Semantic (if present).** Search for defined events / named elements:
```
discover_org_context:
  queries: ["{tool1_name}", "{tool2_name}", "copy", "paste", "submit",
            "click", "field", "search", "ticket", "deal", "task"]
# For each relevant hit → build_metric(single_number) → compute. PROVISIONAL
# until validated against Tier B.
```

**Tier B — Raw aggregate (always available, naming-independent).** Reproduce the
same signals from autocaptured events. Use the `metric_definition` escape hatch
to pin the raw event type if the NL layer mis-maps.
```
# workflow/button clicks → raw selector paths (CAP top ~15–20 by frequency)
build_metric: "clicks grouped by element where URL domain contains {domain}" (top_n)
# field population / data entry
build_metric: "change events where URL domain contains {domain}" (single_number)
# form submissions
build_metric: "clicks on submit elements where URL domain contains {domain}" (single_number)
#   (or build_funnel: "... then submitted the form")
# search/lookup → navigations to the search path (from the 1.1 path breakdown)
build_metric: "page views where URL path contains search AND URL domain contains {domain}" (single_number)
# frustration proxy → reuse Phase 1.3 discover_groups (rage/dead/error); do not rebuild
# unnamed pages are fine: get_pages returns learned pages; URL paths need no names
```
Cross-check A vs B; on material divergence, **trust B** and note it. Keep B
bounded (few signal types per team, selectors capped).

**Tier C — Ground-truth confirmation (automatic, reuse-only).** Don't read
transcripts here. Emit a **confirm-list** (raw selector/event + frequency + tool
+ question) for any high-frequency signal whose *meaning* is ambiguous, and hand
it to Phase 2.2 — its `session-context` subagent already reads the transcript, so
confirmation rides along at ~0 marginal cost. Empty confirm-list (clean org) →
Tier C does nothing.

---

# Team Archetype Patterns (reference library — select & adapt per confirmed team)

The sections below are **not a fixed roster of teams to run.** They are a
reference library of query patterns indexed by team *function*. The actual teams
— their **names and count** — come from the user's Phase 0.4 confirmation
(validated user-property values or tool-cluster groupings).

**How to use this library:**
1. Run **exactly one pass per team confirmed in Phase 0.4** — no more, no fewer.
2. For each confirmed team, pick the archetype below whose **function** is
   closest, and **adapt its queries to that team's actual tools** (from the
   Phase 0.1 inventory). The tool names in each archetype are examples.
3. If a confirmed team matches **no** archetype (e.g., Claims Adjusters, Clinical
   Coders, Trust & Safety), use the **Custom / Other Team** archetype below —
   which is just the Universal Queries + "Adapting to Unknown Tools" pattern.
4. Always use the **user-confirmed team name** for labels/ledger/deliverable —
   never the archetype's example name.

---

## Archetype: Customer Support (example)

Primary tools: Zendesk, Intercom, Freshdesk, ServiceNow, or similar.

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

This pattern applies to **every** team. People-scoped teams should select by
segment (the right people in the tool); tool-cluster teams select by metric.

```
# PEOPLE-SCOPED (preferred): intersected who+what segment → the team's members
# actually working in the tool
build_segment:
  query: "users where {property}={value} who visited {support_tool_domain}"
  start_date: {analysis_start}
  end_date:   {analysis_end}
→ get_sessions(segment_id, limit=10)
# (or reuse the 0.4.2 membership segment for a whole-day, cross-tool view)

# TOOL-CLUSTER (fallback): anyone active in the support tool
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

## Archetype: Sales Operations (example)

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

## Archetype: RevOps / Deal Desk (example)

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

## Archetype: Engineering Ops (example)

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

## Archetype: Marketing Ops (example)

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

## Archetype: Custom / Other Team (use when no archetype above fits)

For any confirmed team whose function doesn't match the archetypes above (e.g.,
Claims Adjusters, Clinical Coders, Trust & Safety, Underwriting, Logistics). Do
not force-fit it into an existing archetype — build the pass from the universal
patterns instead.

### Volume Queries

```
# Per-tool path breakdown for each domain in the team's stack (from Phase 0.1):
build_metric:
  query: "page views grouped by URL path where URL domain contains {domain}"
  output_type: top_n
  start_date: {analysis_start}
  end_date:   {analysis_end}
→ (people-scoped) update_metric(..., segment_id={team_segment}) → compute_metric
# Plus the Universal "Per-Tool Volume Baseline" trio: focused time, unique users.
```

### Interaction Signal Queries

```
# Derive probe terms from the team's actual tools + the generic manual-work verbs:
discover_org_context(queries=["{tool1_name}", "{tool2_name}", "copy", "paste",
                              "submit", "field", "search", "export", "approve"])
# For each relevant defined event found, build a single_number count metric.
```

### Session Selection

Use the Universal "Session Selection" pattern (people-scoped → intersected
who+what segment; tool-cluster → metric_id), substituting the team's primary
tool domain.

### What to Look For in Sessions

The tool-agnostic checklist applies to any team: repetitive data entry,
copy-paste chains between tools, cross-tool context switching (long pauses),
duplicate lookups, manual report/status assembly, and auth/SSO friction. Let the
session reveal the workflow regardless of whether you recognize the tools.

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
| Metric ID | Query | Output Type | Window | Segment ID | Time Property | Signal Name | Blueprint Role | Significance | Baseline Value |
|-----------|-------|-------------|--------|------------|---------------|-------------|----------------|--------------|----------------|
| m-abc123  | count of Copied Comment Log events | single_number | 2026-04-01–2026-04-30 | seg-supp | N/A | Copied Comment Log | Primary signal for #1 Response Assembly | Directly measures copy-paste volume that AI drafting eliminates | 2,847 |
| m-def456  | SUM focused time (builtInProperty 28, escape hatch) where domain contains zendesk | single_number | 2026-04-01–2026-04-30 | seg-supp | focused | Zendesk Engaged Time | Baseline context | Frames total support tool time investment | 412h |
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
