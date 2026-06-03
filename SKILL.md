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
| 0 (opt) | Instrumentation Pre-Flight — spec pages/elements/events to configure first | `instrumentation-spec` skill, `WebSearch` |
| 0.0 | Analysis Window — pin one absolute date range for the whole run | (none — set dates once) |
| 0 | Orientation — discover tools, users, scope teams into segments | `build_metric`, `discover_org_context`, `build_segment` |
| 0.5 | Metric Persistence — save/reference metrics + segments for benchmarking | `get_metric`, `get_segment` (if referencing) |
| 1 | Quantitative Discovery — volumes, signals, friction (people-scoped) | `build_metric`, `compute_metric`, `update_metric`, `discover_org_context`, `discover_groups` |
| 2 | Session Deep Dives — behavioral evidence | `build_segment`, `get_sessions`, `get_session_events` (via subagents) |
| 3 | Synthesis — score and rank candidates | Analysis of Phase 1 + 2 outputs |
| 4 | Deliverable — generate HTML report | Write using report-template.html |
| 5 | Post-Implementation Measurement — before/after proof | `get_metric`, `compute_metric`, `get_sessions` |

---

## Phase 0 (optional): Instrumentation Pre-Flight

Goal: When an org is willing to configure FullStory before the analysis, raise
signal quality by defining tool-aware pages, named elements, and defined events
up front — so the analysis runs on high-precision, action-level events instead
of URL-path inference.

Trigger this when a user names a tool (or team) they want to evaluate and asks
what to configure/instrument first, or when an upcoming analysis would benefit
from richer signal on a specific tool.

Use the **`instrumentation-spec`** skill (see [instrumentation-spec.md](instrumentation-spec.md)).
It researches the tool's real workflows, then produces a single tabbed HTML build
sheet (Overview + one tab per tool + a team rollup) telling a FullStory admin
exactly which pages, named elements, defined events, and custom properties to
create in Data Studio. Note the handoff is human — the analysis tooling can
discover and query definitions but cannot create them. After the admin configures
the spec and ~2-4 weeks of capture accumulate, proceed to Phase 0 Orientation;
the new definitions surface via `discover_org_context` and become primary signal.

Reference example: `instrumentation-specs/servicenow.html`.

---

## Phase 0.0: Establish the Canonical Analysis Window

Goal: Lock a single, **absolute** date range that every metric in this run is
built and computed against — so the numbers in the deliverable stay reproducible
forever and match what a stakeholder sees when they open a linked metric later.

Why this matters: relative presets (`last_30_days`, `this_month`) are saved into
the metric definition and **slide forward over time**. A metric built on
`last_30_days` will show a different number every day it's reopened, so the
deliverable's numbers drift out of sync with the linked FullStory metrics within
hours. Relative windows can also make a single run internally inconsistent if it
crosses midnight. Pinning one absolute window fixes both problems.

### 0.0.1 Compute the Window Once

At the very start of the engagement, establish:

- **`analysis_start`** and **`analysis_end`** — explicit ISO 8601 dates
  (e.g., `2026-05-02` to `2026-06-01`). Default to a trailing 30-day window,
  but **end it ~1 day before today** so late-arriving/still-ingesting events
  don't cause the window to keep shifting as data settles.
- **`as_of`** — the timestamp (with timezone) at which the analysis was
  computed (e.g., `2026-06-02 09:00 PT`).

Record all three immediately — they thread through every later phase (ledger,
metric builds, and the deliverable header).

### 0.0.2 Use Absolute Dates Everywhere That Persists

For **every** `build_metric` and `compute_metric` call whose result is shown in
the deliverable, recorded in the ledger, or linked for verification:

- Pass `start_date=analysis_start` and `end_date=analysis_end` (ISO 8601).
- Do **NOT** pass `time_range` (the relative preset) — it would overwrite the
  pinned window with a sliding one.

Relative presets are acceptable **only** for throwaway exploration whose result
is never linked, recorded, or quoted in the deliverable.

### 0.0.3 Verify the Metric URL Honors the Saved Window

The deliverable links metrics via
`https://app.fullstory.com/ui/{org-id}/metrics/create/{metric_id}`. Before
relying on these links, spot-check **one** metric: open the link and confirm
the date range it renders matches `analysis_start`–`analysis_end` and the value
matches the deliverable.

- If it matches → the saved absolute window is honored; proceed.
- If the UI defaults to a different (e.g., relative) window → append the date
  range to the URL as query parameters so the link opens pinned. Confirm the
  exact parameter names against a live metric URL first (do not assume them),
  and document the working URL format in the ledger so all links use it.

---

## Phase 0: Orientation

Goal: Build a data-driven inventory of the org's tool stack and agree on scope.

### 0.1 Discover the Tool Inventory

Goal: a **complete** list of the hosts (applications) the org touches — this is
the orientation to the tech stack. Volume/time is secondary here (covered in 0.3
and Phase 1). Do NOT hardcode a list of tools; query the org's actual captured
data. No dependency on page definitions or org configuration maturity.

**Step 1 — Build with the correct host dimension.**
Natural-language "URL host" is unreliable — it often maps to the *full URL*
dimension (`DIMENSION_URL`, one row per URL incl. path/query), which fragments
each tool across dozens of rows. Force the host rollup instead:

1. `build_metric`: query="page views grouped by URL host",
   output_type="top_n", start_date=analysis_start, end_date=analysis_end
   (absolute window from Phase 0.0 — never `time_range`).
2. Inspect the returned definition. The dimension MUST be
   `dimensionSettings.dimension.builtInProperty = "DIMENSION_URL_HOST"`. If it
   came back as `DIMENSION_URL` (full URL), refine it:
   `update_metric(refinement="group by URL host instead of the full URL")`,
   which reliably yields `DIMENSION_URL_HOST`.

**Step 2 — Pull the COMPLETE list in one call (beat the 50-row cap).**
The builder/UI default caps `top_n` at 50 rows, but the server honors a higher
limit when you compute a raw definition. Take the `DIMENSION_URL_HOST`
definition, set `dimensionSettings.limit` high (start at **200**), and run it
through the `compute_metric` **`metric_definition`** escape hatch. This returns
the full host list — down to single-view hosts — in a single call.

**Step 3 — Coverage check (completeness guarantee + truncation detector).**
Every `top_n` result includes a `total` (page views across ALL hosts, not just
returned rows). Compute:

```
coverage % = sum(returned row values) ÷ total × 100
```

- If coverage = 100% → every host captured; proceed.
- If the returned row count equals `limit` AND coverage < 100% → the list is
  truncated. Raise `limit` (e.g. 500) and recompute, or use the pagination
  fallback below. Record the final coverage % to report in the deliverable.

**Step 4 — Normalize for the inventory, keep raw hosts.**
For the human-facing tool inventory, collapse hosts to their **registrable
domain** so each tool is one line (`web.outreach.io`, `sidebar.outreach.io`,
`accounts.outreach.io` → `outreach.io`; `fullstory.my.salesforce.com` +
`*.lightning.force.com` → Salesforce). Keep the raw host list too — some
subdomains are genuinely distinct tools.

**Step 5 — Keep ALL hosts (do not exclude noise).**
Do NOT drop auth/SSO, CDN, or internal hosts — they are part of the org's tool
reality and matter to the evaluation (e.g. time lost to SSO/identity friction).
You MAY *label* categories for readability (e.g. tag `*.okta.com` as
"SSO/Identity", `vertexaisearch…` as "AI/Search"), but every host stays in the
inventory.

**Pagination fallback (only if a single high limit can't capture everything).**
There is no offset/page parameter, so "paging" means excluding what you've
already seen. In the raw definition, add a host-exclusion dependency:

```json
"deps": [
  { "url": { "notEquals": { "value": [
      { "category": "URL_HOST", "value": "host1.com,host2.com,..." }
  ] } } }
]
```

(hosts are a single comma-separated string). Capture a page, append its hosts to
the exclusion string, recompute, and repeat until coverage reaches the target.

**Notes.**
- Each `metric_definition` compute saves an *unnamed* metric in the org. Note
  the ledger entry so it can be named/cleaned up later; don't leave them
  unexplained.
- The saved metric's link opens in the FullStory UI, which renders only the
  **top 50** rows even when the saved `limit` is higher. When linking the host
  inventory for verification, add a tooltip/footnote: the analysis lists the
  full set; the FullStory UI shows the top 50 due to a platform display cap.
  (The analysis itself is conducted outside FullStory; the link is for verifying
  the underlying metric, not for reproducing the full table.)

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

**Enumerate the team values.** Once you've picked the best grouping property
(e.g., `department`), you need its distinct **values** to build segments. There
is no "list all values of a property" tool, so:
- Read candidate values from the `discover_org_context` results / schema sample, AND
- Confirm the value list with the user ("I see department values like Support,
  Sales, Engineering — which map to the teams you want analyzed?").
- Lock a `{team → property=value}` mapping. This drives segment creation (0.4.2).

**Set the scoping mode (per team, not global).** This is the hybrid pivot:
- If a team maps cleanly to a user-property value → mark it **people-scoped**.
  Its Phase 1/2 metrics and sessions will be scoped to a user segment.
- If a team has no usable property (messy/absent values) → mark it
  **tool-cluster** (fallback): its metrics stay URL/domain-based and the
  deliverable carries the inferred-grouping caveat for that team only.
- A messy property for one team does NOT force the others into fallback.

### 0.3 Enrich with Engaged Time

Enrich the inventory with time spent. Include **all** hosts/domains from 0.1 —
do not exclude auth/SSO, CDN, or internal hosts (time lost to SSO/identity
friction is itself a finding). For efficiency, enrich the registrable domains
from 0.1 Step 4 (so subdomains roll up into one tool), working top-down by
volume so the highest-impact tools are quantified first.

**Use focused time as the primary measure (not active time).**

- **`Page focused time (new)`** — time the tool's tab was focused/foreground.
  This is the fairest cross-tool measure because it counts *engaged attention*
  including reading, reviewing, and watching (support reading tickets, reps
  listening to Gong calls, analysts reading dashboards) — work that
  `Page active time (new)` (interaction-only) systematically undercounts.

> **CRITICAL — the NL builder cannot target focused time.** `build_metric`
> silently maps the phrase "Page focused time (new)" onto the **active**
> property, and the refiner falls back to a plain count. The ONLY reliable way to
> measure focused time is the `compute_metric` **`metric_definition` escape
> hatch** with the numeric property id below. Never trust a "focused" duration
> from an NL build without verifying `property.builtInProperty`.

**Page-timing property enums** (global proto enums — portable across orgs. Event
node must be `visitedPage`; use `NUMERIC_AGGREGATION_METHOD_SUM` for totals):

| Property | `builtInProperty` value |
| --- | --- |
| `Page focused time (new)` (engaged) | `28` (numeric) |
| `Page active time (new)` (interaction-only) | `"NUMERIC_PROPERTY_PAGEVIEW_ACTIVE_DURATION"` |
| `Page visible time (new)` (on-screen) | `"NUMERIC_PROPERTY_PAGEVIEW_TOTAL_DURATION"` |

**Recipe — clone-and-swap (preserves the correct scope).** The escape hatch
needs the full filter tree, so derive focused time from a correctly-scoped
metric rather than hand-building filters:

1. Build the scoped metric via NL using **active** time (NL maps active
   correctly): `build_metric` query="total Page active time (new) where URL
   domain contains {domain}", output_type="single_number",
   start_date=analysis_start, end_date=analysis_end (pinned window from Phase
   0.0 — no `time_range`). For people-scoped teams, attach the team segment via
   `update_metric` segment_id. `compute_metric` → **active hours** for that scope.
2. Take the returned `metric_definition`, set
   `metric.single.aggregation.numeric.property.builtInProperty = 28`, leave the
   event filter tree, attached segment, and window unchanged, and `compute_metric`
   with that `metric_definition` → **focused (engaged) hours** for the same scope.
3. (Optional on-screen denominator) repeat step 2 with property
   `"NUMERIC_PROPERTY_PAGEVIEW_TOTAL_DURATION"` → **visible hours**.

Use focused (engaged) hours as the headline per-tool time measure.

**Primary tool only — the work-type ratio.** For each team's **primary tool**
(not every tool), compute the **work-type ratio = active ÷ focused** from the two
numbers above:
   - **High ratio (~0.7+)** → input-heavy (typing, clicking, data entry) →
     points toward *auto-population / data-entry* automations.
   - **Low ratio (~0.4-)** → read/review-heavy (focused but little input) →
     points toward *surface-info-inline / summarization / AI-assist* automations.
   This ratio feeds the Phase 3.2 automation-level classification.

**Units.** These properties return **milliseconds**. Convert before reporting:
hours = value ÷ 3,600,000 (e.g. 24,195,368,337 ms ≈ 6,721 h). The work-type
ratio is unitless (ms ÷ ms), so compute it on raw values.

**Verify before trusting numbers.** Confirm each returned definition's
`property.builtInProperty` is the one you intended, and sanity-check the nesting
**visible ≥ focused ≥ active** (on-screen ⊇ tab-focused ⊇ actively-interacting).
A focused or active value of 0 org-wide means the time layer is unpopulated for
that org — fall back to page-view volume as the time proxy and note the caveat.

### 0.4 Present and Scope

Present the ranked tool inventory to the user with page views + active
time. If Phase 0.2 revealed user properties with team/department/role
data, show team groupings grounded in actual user data:

> "34 users with department=Support spend 80% of time in Zendesk"

If no meaningful user properties exist, recommend team groupings based
on observed tool clusters as a fallback. **Derive the clusters from THIS
org's own Phase 0.1 inventory — do not match against a predefined list.**

How to derive clusters from the org's data:
1. Start from the actual hosts/domains in the Phase 0.1 inventory (this org's
   real stack, whatever it is — niche, internal, non-English, or industry-
   specific tools included).
2. Group hosts that are **co-used by the same users in the same sessions**
   (tools that show up together in a user's day belong to the same workflow).
   Use `discover_org_context` / session sampling if co-usage isn't obvious
   from the inventory alone.
3. Name each cluster by the **function** of its dominant (highest-volume) tool,
   in language that fits the org. If a cluster's purpose is unclear, present it
   by its tools and let the user name it.
4. Never invent a team the data doesn't support, and never drop a high-volume
   tool just because it doesn't fit a familiar pattern.

The table below is **illustrative only — examples of how common SaaS tools
tend to cluster into functions. Do NOT treat it as a lookup table or match the
org's tools against it.** It exists to show the *shape* of a good cluster, not
to enumerate valid answers:

| Example Tool Cluster (illustrative) | Example Team Name |
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

- Teams to analyze (user-confirmed names + which tools map to each, plus the
  **primary tool** for each team — the one that gets the tool-only adoption
  metric in Phase 1)
- Scoping mode per team (people-scoped vs tool-cluster, from Phase 0.2)
- Analysis window — confirm the absolute `analysis_start`–`analysis_end`
  dates pinned in Phase 0.0 (default: trailing 30 days ending ~1 day ago)
- Number of session deep dives per team (default: 3)

**The confirmed dive count is a hard commitment, not a target.** Once the user
confirms (or accepts the default of) N dives per team, run exactly N per team in
Phase 2 (a team that genuinely has no qualifying sessions — e.g. zero tracked
activity — is the *only* allowed exception, and it must be called out as a
caveat, not silently dropped). **Never reduce N on your own** for token budget,
run size, effort, latency, or because the engagement feels like a "test." Note
that `session-context` subagents run in isolated contexts and can be launched in
parallel, so a large `teams × N` is cheap to your own context — volume is not a
reason to trim. If `teams × N` still looks too expensive to execute, **stop and
tell the user the exact total and ask them to pick a smaller N explicitly** —
do not pick one for them. If you ever deviate from the confirmed N for any
reason, surface it prominently in your run summary and in the deliverable's
methodology, never as a silent change.

### 0.4.2 Build Team Segments (people-scoped teams only)

For every team marked **people-scoped** in Phase 0.2, create a reusable user
segment that defines the team's membership:

```
build_segment(query="users where {property} = {value}",
  name="{prefix} {Team Name} Members",
  start_date=analysis_start, end_date=analysis_end)   # pin the window (Phase 0.0)
```

- Record the returned `segment_id` and definition in the ledger (Phase 0.5).
- This segment is the **WHO** for the team. Phase 1 metrics attach it
  (`update_metric`) and Phase 2 builds intersected who+what segments from it.
- For **tool-cluster** teams, skip this — there is no segment; their metrics
  remain URL/domain-scoped.

These segments are reusable assets: they power per-team scoping now and
before/after benchmarking in Phase 5.

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
Analysis Window: {analysis_start} – {analysis_end}  (absolute, pinned in Phase 0.0)
Computed As-Of: {as_of}  (timestamp + timezone)
Metric URL Format: {verified_url_format}  (from Phase 0.0.3 — note if query params needed)
Teams: {team1} ({people-scoped|tool-cluster}), {team2} (...), ...
Created: {timestamp}

## Segments (people-scoped teams)

| Segment ID | Team | Definition (property = value) | Window | Scoping Mode |
|------------|------|-------------------------------|--------|--------------|

## Metrics

| Metric ID | Query | Output Type | Window | Segment ID | Time Property | Signal Name | Blueprint Role | Significance | Baseline Value |
|-----------|-------|-------------|--------|------------|---------------|-------------|----------------|--------------|----------------|
```

The **Window** column records the absolute `start_date`–`end_date` each metric
was built with. It should equal the analysis window for every metric; a row that
differs is a red flag that a relative preset leaked in. The **Segment ID** column
records which team segment a metric was scoped to (blank = unscoped/tool-cluster
or org-wide context metric).

After each `build_metric` → `compute_metric` sequence in Phase 1, append
a row to the ledger:

- **Window**: the absolute `start_date`–`end_date` the metric was built with
  (must equal the Phase 0.0 analysis window)
- **Segment ID**: the team segment attached to this metric (blank if the metric
  is org-wide context or a tool-cluster team's URL-scoped metric)
- **Time Property**: `focused` (`Page focused time (new)`), `active` (`Page
  active time (new)`, primary tool only — for the work-type ratio), or `N/A`
  (non-time metric)
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

Run for each team. Queries use URL-based filtering (not page definitions) for
tool/event scoping, so completeness doesn't depend on org configuration maturity.

**Every** `build_metric` / `compute_metric` call in this phase must pass the
absolute `start_date=analysis_start` and `end_date=analysis_end` from Phase 0.0
(never the relative `time_range` preset), so the saved metrics — and the links
to them in the deliverable — stay pinned to the reported window.

### 1.0 Apply the team's scoping (hybrid)

The WHAT (tool/URL/event) comes from the metric; the WHO comes from the team's
scoping mode (Phase 0.2 / 0.4.2):

- **People-scoped team** → after building a metric, attach the team segment so
  the result counts only that team's members:
  ```
  build_metric(..., start_date=analysis_start, end_date=analysis_end)
    → update_metric(metric_id, segment_id={team_segment})   # returns a new metric_id
    → compute_metric(new_metric_id)
  ```
  This yields the people × tool intersection ("Zendesk page views **by Support
  members**"). Record the `segment_id` in the ledger row.
- **Tool-cluster team** (fallback) → skip the attach; compute the URL-scoped
  metric as-is (today's behavior). Carry the inferred-grouping caveat for this
  team in the deliverable.

**Run exactly one pass per team confirmed in Phase 0.4 — no more, no fewer.**
The number and names of teams come from that confirmation (validated user-property
values or tool-cluster groupings); they are never a fixed list. See
[query-playbook.md](query-playbook.md) for detailed query sequences. Its per-team
sections are **archetype examples keyed to function** (Support, Sales Ops, etc.),
not required teams: for each confirmed team pick the closest archetype and adapt
its queries to that team's actual tools, and if none fit, use the Universal
Queries + "Adapting to Unknown Tools" pattern.

### 1.1 Volume & Time

Use **engaged time = `Page focused time (new)`** as the primary time measure
throughout. Measure it with the Phase 0.3 **clone-and-swap recipe**
(`metric_definition` escape hatch, `builtInProperty: 28`) — the NL builder
silently substitutes the active property for focused, so never measure focused
via a raw NL query.

**A. Cross-tool team view (people-scoped teams — the headline workforce view).**
Reuse the hardened `DIMENSION_URL_HOST` breakdown from Phase 0.1, then swap its
aggregated property to focused time (`builtInProperty: 28`) via the escape hatch,
attach the team segment, and compute → the team's **complete focused-time
distribution across their entire stack** (e.g., "Support: 58% Zendesk, 14%
Salesforce, 9% SSO"). This shows where a team's hours actually go, not just one
tool's traffic.

**B. Per-tool volume & time** — for each tool in the team's stack:
- `top_n`: "page views grouped by URL path where domain contains {domain}"
- `single_number`: "unique users where URL domain contains {domain}"
- **focused (engaged) time**: build the active-time metric for the domain via NL,
  then clone-and-swap its property to `builtInProperty: 28` (Phase 0.3 recipe) and
  `compute_metric` the swapped definition.

For people-scoped teams, attach the team segment (1.0) so these reflect the
team's members. For tool-cluster teams, leave them URL-scoped.

**C. Primary tool only — adoption + work-type ratio (people-scoped teams).**
For each team's **primary tool** (not every tool), also compute:
- *Tool-only adoption*: the *unscoped* version of the focused-time/volume metric.
  team-scoped ÷ tool-only = the team's share of that tool ("Support = 80% of all
  Zendesk").
- *Work-type ratio*: active ÷ focused on the primary tool (both from the Phase
  0.3 clone-and-swap: active = the NL-built metric, focused = same definition with
  `builtInProperty: 28`) — high = input-heavy, low = read/review-heavy. Feeds
  Phase 3.2.

### 1.2 Interaction Signals

Goal: quantify the manual-work signals — copy/paste, form submissions, field
changes, search/lookup, workflow-trigger clicks. The challenge: not every org
has clean, semantic instrumentation. Defined events and named elements may be
sparse, missing, or **misnamed/misleading**. They are a convenience layer on top
of FullStory's complete raw, autocaptured event stream — so we never depend on
them being present or correct.

**Guiding principle — names are hints, raw behavior is truth.** Use any defined
events/named elements only to know *where to look*. Confirm every signal against
raw frequency (and, where meaning is unclear, session replay) before counting
it. If a named event's volume diverges from the raw click/selector volume, trust
the raw. This makes 1.2 robust to absent or wrong naming.

Work down these tiers, cross-checking upward. For people-scoped teams, attach
the team segment (1.0) to every metric so counts reflect the team's members.

**Tier A — Semantic (use if present).** `discover_org_context` with queries
derived from the team's actual tools (NOT a hardcoded list):
```
discover_org_context(queries=["zendesk", "ticket", "submit", "copy", ...])
```
For each relevant defined event / named element found, build a `single_number`
metric to count occurrences. Treat these counts as **provisional hints** until
validated against Tier B.

**Tier B — Raw aggregate (always available, naming-independent).** Reproduce the
same signals directly from autocaptured events via `build_metric` (use the
`metric_definition` escape hatch to pin the raw event type if the NL layer
mis-maps). Map each target signal to a raw query:
- Workflow-trigger / button clicks → **clicks grouped by element** (returns raw
  CSS-selector paths even with no friendly name). **Cap to the top ~15–20
  selectors by frequency.**
- Field population / data entry → **change / input events** on the tool's domain
- Form submissions → **clicks on submit-type elements**, or a `build_funnel`
  sequence ending in the post-submit navigation
- Search/lookup → **navigations to the tool's search path** (from the 1.1 path
  breakdown)
- Frustration (manual-workaround proxy) → already covered by the Phase 1.3
  `discover_groups` proactive metrics (rage/dead/error clicks) — reuse, don't
  rebuild
- Unnamed pages are fine: `get_pages` returns **learned** pages too, and the
  URL-path breakdown from 1.1 needs no page names at all.

Cross-check Tier A vs Tier B: where a named event's count materially diverges
from its raw equivalent, **trust Tier B** and note the discrepancy. Keep Tier B
bounded — only the few signal *types* that matter per team, top-N selectors
capped as above.

**Tier C — Ground-truth confirmation (automatic, reuse-only, no extra cost).**
Aggregates give *frequency* but not *meaning* — a high-frequency unnamed selector
is a strong signal whose purpose still has to be confirmed. Rather than read
transcripts here (expensive), **emit a "signals to confirm" list** and hand it to
Phase 2: the Phase 2.2 `session-context` subagent already reads each selected
session's full transcript in an isolated context, so confirming these signals
rides along on a read that happens anyway (marginal cost ≈ 0). Only populate the
confirm-list when there is genuine ambiguity (weak/misleading Tier A naming **and**
high-frequency selectors Tier B can't interpret); in a cleanly-named org it is
empty and Tier C does nothing.

**Confirm-list (output of 1.2 → input to Phase 2.2):** for each signal needing
ground-truth, record: the raw selector/event, its Tier-B frequency, the tool, and
the question to resolve (e.g., *"selector `div.btn-xyz` clicked 4,800× on
zendesk.com — what action is this and is it manual data transfer?"*). Skip if empty.

### 1.3 Friction & Error Discovery (always-on)

Use `discover_groups` to surface the org/team's top frustration and error
groups — a frustration cluster is frequently a manual workaround signal and a
strong automation candidate. Run this for **every** team (always-on, lightweight):

- **People-scoped team**: `discover_groups(segment_id={team_segment}, limit=5,
  start_time=analysis_start, end_time=analysis_end)` → the team's top friction.
- Also run `discover_groups(domain={primary_tool_host}, limit=5, ...)` for the
  team's **primary tool** to catch tool-specific friction regardless of who.
- **Tool-cluster team**: use `domain={primary_tool_host}` scoping (no segment).

Capture the returned groups (and each `metric_url`) as candidate evidence. Feed
them into Phase 3 alongside the quantitative signals. Keep it to the top ~5 per
scope — this is a fast supplementary signal, not an exhaustive audit.

### 1.4 Behavioral Patterns

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

### 1.5 Document Findings

Record a team-level summary plus per-tool detail.

Team-level (people-scoped teams):
```
Team: {name}
Scoping Mode: people-scoped | tool-cluster
Segment ID: {segment_id or N/A}
Members (segment size): {count}
Cross-Tool Distribution: {host → %/focused-hours across the team's whole stack}
Primary Tool Adoption: {team share of primary tool traffic, from 1.1C}
Primary Tool Work-Type Ratio: {active ÷ focused} ({input-heavy | read-heavy})
Top Friction Groups: {discover_groups results + metric_urls}
```

Per tool:
```
Tool: {name}
Domain: {domain}
Scope: team-segment {segment_id} | tool-cluster (URL only)
Page Views: {count}
Engaged Time (focused): {hours}h   # Page focused time (new)
Active Time (primary tool only): {hours}h   # Page active time (new)
Users: {count}
Top Paths: {path → count list}
Notable Signals: {signal → count, each tagged with source: Tier A (named event) |
  Tier B (raw selector/event); note any A↔B divergence and which was trusted}
Signals to Confirm (Tier C): {raw selector/event, frequency, question — or "none"}
Anomalies: {anything unexpected}
```

---

## Phase 2: Session Deep Dives

Goal: Observe actual employee behavior to identify manual workflow loops,
friction points, and automation opportunities that quantitative data alone
cannot reveal.

### 2.1 Session Selection

Goal: watch the **right people** doing the work, not just anyone who touched the
tool. `get_sessions` accepts *either* a `metric_id` *or* a `segment_id` (not
both), so scope by team via a segment:

- **People-scoped team** — build an **intersected who+what segment** so you get
  the team's members *in the relevant tool*, then pull sessions:
  ```
  build_segment(query="users where {property}={value} who visited {primary_domain}",
    start_date=analysis_start, end_date=analysis_end)
    → get_sessions(segment_id, limit=...)
  ```
  This is more accurate than today's "anyone who hit the tool." You can also
  reuse the team membership segment from 0.4.2 directly for a whole-day,
  cross-tool view.
- **Tool-cluster team** (fallback) — `get_sessions(metric_id=...)` against a
  metric targeting the primary tool (today's behavior).

Select exactly the **confirmed number of dives per team from Phase 0.4.1**
(default: 3) — this is a hard commitment. Do **not** reduce it here for token
budget, run size, or a "representative sample" shortcut; `session-context`
subagents are isolated and parallelizable, so running the full `teams × N` is
cheap to your own context. The only acceptable shortfall is a team with no
qualifying sessions (e.g. zero tracked activity), which must be flagged as a
caveat in the deliverable — never dropped silently. If the total genuinely needs
to shrink, return to the user and have them confirm a smaller N (per Phase
0.4.1). Aim for:
- **High page count** (sustained work sessions, not drive-by visits)
- **Multiple tool domains** if possible (cross-tool switching)
- **Presence of friction signals** (rage clicks, dead clicks, mouse thrash) —
  the Phase 1.3 `discover_groups` output is a good guide to which sessions to pull.

If the first batch doesn't yield good candidates, refine the segment
(`update_segment`) or try a metric targeting a specific signal (copy-paste
events, workflow triggers). Note `get_sessions` caps at `limit=50`.

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
>
> For every moment you cite, return BOTH timestamp forms so the deliverable can
> deep-link to it: (a) `absolute_ms` — the moment's **absolute Unix-epoch
> milliseconds** (session start epoch + the offset from session start; the raw
> events carry the absolute time), and (b) `mm:ss` — the human-readable offset
> from session start. Pair each with a one-sentence note of what is happening.

**Tier C confirmation (reuse-only).** If Phase 1.2 produced a non-empty
"signals to confirm" list for this team, append it to the subagent task above so
the transcript read — which happens regardless — also resolves it:

> Additionally, resolve these specific raw signals observed in aggregate. For
> each: confirm what the user action actually is, whether it represents the
> suspected manual work, and roughly how often/long it occurs in this session.
> {paste the 1.2 confirm-list — selector/event, frequency, tool, question}

Skip this addendum entirely when the confirm-list is empty (cleanly-named org).
Do not pull extra sessions for it — confirm against the sessions already
selected in 2.1.

### 2.3 Extract Evidence Clips

From each session analysis, identify the 2–3 most compelling moments that
illustrate manual work a machine should handle. Record:
- **Deep-link URL** (jumps straight to the moment — no manual scrubbing):
  `https://app.fullstory.com/ui/{org-id}/session/{device-id}:{session-id}:{absolute_ms}`
  where `{absolute_ms}` is the moment's **absolute Unix-epoch milliseconds** from
  the 2.2 subagent. Appending `:{ms}` to the session URL is FullStory's documented
  deep-link format; without it the link only opens session start.
- Display timestamp `mm:ss` (offset from session start) — shown as the visible
  link label; the deep-link `{absolute_ms}` is what the `href` carries.
- **Note** — a one-sentence description of what's happening *at that exact
  timestamp* (from the 2.2 subagent). This renders inline next to the clip so the
  reader sees the link, the moment, and the explanation together.
- Which automation candidate it supports

These become the "Watch It Happen" links in the deliverable — each a direct
deep-link to the moment with its note.

---

## Phase 3: Synthesis & Scoring

See [scoring-rubric.md](scoring-rubric.md) for the detailed methodology.

### 3.1 Identify Automation Candidates

Draw candidates from three sources observed across Phases 1 and 2:
1. **Quantitative signals** — high-volume URL paths, interaction-signal metrics.
2. **Session observations** — repetitive loops, copy-paste chains, manual entry.
3. **Friction & error groups** — the `discover_groups` output from Phase 1.3.
   A recurring frustration/error cluster usually marks a manual workaround;
   treat each material group as a candidate and cite its `metric_url` as evidence.

For each manual workflow, create a candidate. For people-scoped teams, prefer
the team-segment-scoped metric values (the team's own volume) as the primary
signal; note the scope so estimates aren't inflated by other departments.

| Field | Source |
|-------|--------|
| Name | Descriptive workflow name |
| Signal from Data | Which Phase 1 metrics support this (note segment scope) |
| Friction Evidence | Related `discover_groups` group(s) + metric_url, if any |
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

**Use the primary tool's work-type ratio (active ÷ focused, Phase 0.3/1.1C) as a
classification heuristic:**
- **High ratio (input-heavy)** → manual typing/clicking/data entry dominates →
  favor **Fully Automated** (auto-populate, sync) and **Process Automated**.
- **Low ratio (read/review-heavy)** → focused attention with little input
  (reading, reviewing, watching) → favor **AI-Assisted** (summarization,
  surfacing info inline, drafted responses) — automate the *reading*, not typing.

This is a heuristic, not a rule: confirm against session evidence before
finalizing the level.

### 3.3 Rank Candidates

Sort all candidates across all teams by:
1. Monthly hours saved (primary — descending)
2. Implementation complexity (secondary — ascending)
3. Automation level (tertiary — Fully Automated first)

### 3.4 Validate Key Metrics

Before presenting results, cross-check the **primary signal metrics**
(the ones that drive time estimates) against known baselines in the org.

**Step 1: Identify validation candidates**

Any metric that is a primary signal for an automation candidate (i.e., the
metric whose volume feeds directly into the hours-saved estimate) MUST be
validated. Supporting/context metrics are lower priority but should be
spot-checked.

**Step 2: Search for existing org metrics**

```
get_metric(regex="{signal_keyword}")
```

For each primary signal, search for named metrics the org may already
track (e.g., "ticket volume", "reply count"). If the org has existing
metrics, compare:

- The agent's computed value vs the org's existing metric value
- The query scope (what's included/excluded)
- The time range (exact date boundaries)

**Step 3: Investigate divergences**

If a metric diverges >20% from a known org baseline:

1. **Check time range alignment** — because this analysis pins an absolute
   `analysis_start`–`analysis_end` window (Phase 0.0), our metrics are
   unambiguous; the common cause of divergence is the *org's* metric using a
   relative or different window. Confirm the org baseline's exact start/end
   dates against ours before treating a delta as real.
2. **Check event counting method** — is the org counting unique sessions
   with the event, or total event occurrences? These can differ 3-5x.
3. **Check filter scope** — does the org's metric filter by user segment,
   page, or other conditions the agent's query doesn't include?
4. **Rebuild if needed** — if the divergence is explained by scope
   differences, adjust the agent's query to match or document the
   difference explicitly.

**Step 4: Record validation status**

For each primary signal metric, record in the ledger (or in working notes
if ledger is disabled):

| Metric | Agent Value | Org Baseline | Delta | Explanation | Status |
|--------|-------------|-------------|-------|-------------|--------|
| {name} | {value} | {org_value} or N/A | {%} | {why different} | Validated / Adjusted / Flagged |

Metrics with status "Flagged" must be called out in the deliverable with
a note explaining the discrepancy and what the number represents.

**Step 5: Adjust estimates if needed**

If validation reveals the agent's metric was overcounting (e.g., including
bot traffic, test users, or duplicate events), recalculate the hours-saved
estimate with the corrected number. Re-rank if the ordering changes.

**Important**: The goal is NOT to make numbers match perfectly — it's to
understand and explain any differences. A transparent explanation of "our
metric counts X while your dashboard counts Y because of Z" is far more
credible than numbers that don't reconcile.

### 3.5 Backfill Metric Ledger

If metric persistence is enabled (Phase 0.5), go back through the ledger
and fill in the "Blueprint Role" and "Significance" columns now that
automation candidates are identified. Each metric should be tagged with:

- Which opportunity it's a **primary signal** for (the metric that directly
  measures the manual work the automation will eliminate)
- Which opportunity it **supports** (context metrics like total volume,
  engaged (focused) time breakdowns, etc.)
- **Baseline context** for metrics that don't map to a specific opportunity
  but ground the overall analysis

Also add a suggested dashboard name for each metric:
`[{prefix}] {Signal Name}` — e.g., `[WFA-Acme-2026-05] Copied Comment Log`

### 3.6 Build Implementation Roadmap (per team)

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
2. Methodology (always present, always second)
3. One tab per confirmed team, using the exact name the user provided
4. Measurement & Proof (always present, always last)

**Team accent colors** assigned by tab order:
- Team 1: purple (#6c5ce7, #a29bfe)
- Team 2: blue (#3b82f6, #60a5fa)
- Team 3: green (#10b981, #34d399)
- Team 4: amber (#f59e0b, #fbbf24)
- Team 5+: cycle from the beginning

**Analysis Window Stamp** (required, every tab where numbers appear):
Display the absolute analysis window and as-of timestamp prominently and
consistently — e.g., "Analysis window: May 2 – Jun 1, 2026 · computed
2026-06-02 09:00 PT". It must appear in the Overview hero, the Methodology
tab, each team tab subtitle, the Measurement & Proof intro, and the footer,
using the **same absolute string** everywhere. This is what lets a reader
reconcile every linked metric against the report. Never render a relative
phrase like "last 30 days" as the window.

**Overview Tab**:
- Hero with aggregate stats (total hours saved, FTE equivalent, workflow
  count, session count) plus the analysis window stamp
- Cards linking to each team blueprint tab (one card per confirmed team)
- Summary bar by automation level (Fully Automated / AI-Assisted /
  Process Automated)
- Ranked table of ALL candidates across all teams with: rank, team,
  workflow name, automation level, hours saved, primary tools, and
  anchor link to the candidate card
- "Key Moments" section: 8–10 curated session replay links with
  timestamps and descriptions

**Methodology Tab** (always present, second tab):

A concise, customer-facing explanation of how the analysis works. This
tab exists so any stakeholder can understand and challenge the approach.

Contents:

- **How This Analysis Works**: 4-step visual summary
  1. We measured what tools your team uses and how much time they spend
     (FullStory Workforce captures every page view, click, and interaction)
  2. We watched real employee sessions to observe manual workflows
     (session replays — not surveys, not interviews, actual behavior)
  3. We identified repetitive patterns that a machine could handle
     (matching quantitative signals to qualitative observations)
  4. We estimated impact using conservative assumptions you can verify
     (every number traces back to a specific metric and formula)

- **How Teams Are Defined**: One line on scoping. When a team maps to a
  FullStory user property (department/role/team), its numbers reflect *that
  team's actual members* (a user segment); otherwise the team is approximated
  from tool-usage clusters and labeled as such. This tells stakeholders whether
  "the Support team" means real people or "whoever used the support tools."

- **How Time Is Measured**: Define the time metric so the numbers are
  interpretable. "Engaged time" = FullStory's `Page focused time (new)` — time a
  tool's tab was focused/foreground, which counts reading and reviewing, not just
  clicking and typing. For each team's primary tool we also show "active time"
  (`Page active time (new)`, interaction only); the **active ÷ focused ratio**
  indicates whether the work is input-heavy (typing/data entry) or
  read/review-heavy (reading/watching) — which shapes the kind of automation
  recommended.

- **The Estimation Formula**: Show the formula visually:
  `Monthly Hours Saved = Events/Month × Minutes/Event × Eliminability% ÷ 60`
  With a plain-English explanation of each variable.

- **Where the Numbers Come From**: Table explaining each input source:

  | Input | Source | How to Verify |
  |-------|--------|---------------|
  | Event volume | FullStory metric (linked) | Click the metric link in any candidate's audit trail |
  | Time per event | Session replay observation or conservative default | Watch the linked session replay at the cited timestamp |
  | Eliminability % | Based on automation type + workflow characteristics | See rationale in each candidate's audit trail |

- **Conservative by Design**: Callout explaining that estimates use
  conservative defaults, round down, and apply modest eliminability
  percentages. The goal is a floor estimate a skeptic would accept.

- **Validation Status**: Summary of Phase 3.4 validation results:
  - How many metrics were cross-checked against existing org metrics
  - How many matched within 20%
  - Any flagged discrepancies with explanations
  - Statement: "Every primary signal metric that drives an hours-saved
    estimate has been validated or its limitations are documented."

- **How to Challenge a Number**: Step-by-step guide for stakeholders:
  1. Find the candidate card for the opportunity in question
  2. Expand "How We Got Here" to see the full audit trail
  3. Check the metric link — does the FullStory metric match?
  4. Check the session replay — does the observed behavior match?
  5. Check the formula — do the inputs and math check out?
  6. If any input is wrong, recalculate with the corrected value

**Per-Team Blueprint Tabs** (one per user-confirmed team):
- Tab label = the exact team name the user confirmed in Phase 0.5
- **Scoping badge**: indicate how this team was analyzed — "People-scoped
  (segment: {property}={value}, {N} members)" or "Tool-cluster (inferred)".
- Executive callout with key numbers
- **Cross-tool time distribution** (people-scoped teams): chart of where the
  team's hours/page views go across their *entire* stack (from Phase 1.1A),
  e.g. a horizontal bar or donut of host → % of the team's engaged (focused)
  time. This is the headline "where this team's day goes" view.
- **Primary tool** (people-scoped teams): the team's share of total primary-tool
  traffic (from Phase 1.1C, "Support = 80% of all Zendesk") plus the active ÷
  focused work-type ratio (input-heavy vs read/review-heavy).
- Tool volume breakdown (page views, engaged (focused) time, users per tool;
  primary tool also shows active time + ratio). Keep the standard "how to read
  these cards" aside (the three dimensions — paths = *what* they do, users =
  *breadth*, focused time = *depth*) and fill `{volume_scope_note}` with the
  team's scope (segment-scoped vs tool-cluster caveat).
- **"Where Time Goes: {primary_tool}"** — per-tool path breakdown. Build one band
  per major tool (primary first) from the Phase 1.1B path data: a horizontal
  `bar-row` chart of the top URL paths by volume **beside an inline-SVG daily
  trend line** for that tool (the trend gives the report a temporal pulse — daily
  page views across the analysis window). Optionally add a secondary-tool **donut +
  "Top Content" table** where a categorical split tells the story better than paths
  (BI dashboards, call libraries). Use the populated examples in the template;
  replace numbers, labels, and SVG point coordinates with real values.
- **Manual Work Signals** — the Phase 1.2 interaction signals as `bar-row` charts
  (copy/paste, field changes, submits, search), a **Notable Patterns table**
  (Pattern | Volume | Implication), and a headline `stats-3` of the biggest signal
  totals. Tag each signal with its source — Tier A (named event) vs Tier B (raw
  selector) — and note any A↔B divergence per Phase 1.2.
- **Number-anchored insight callouts**: every chart/table gets a one-line takeaway
  callout that ties a *specific figure* to its meaning (e.g. "KB is 0.4% of Zendesk
  views — agents copy from old tickets instead of searching it"). A chart without
  an insight is incomplete.
- **Core Workflow Cycle** (when the team repeats one dominant multi-step cycle
  observed in Phase 2): a **Current State (N steps) vs Future State (M steps)**
  side-by-side step table (future steps use `row-success`). The step-count collapse
  (e.g. 10 → 3) is the single most persuasive exec visual. Omit if no dominant cycle.
- **Friction & errors**: top `discover_groups` groups for this team/primary tool
  (from Phase 1.3), each linking to its FullStory `metric_url`. Frame these as
  candidate evidence ("this frustration cluster signals a manual workaround").
- **Session deep dive cards**: one `session-card` per Phase 2 session that revealed
  a distinctive workflow — a short narrative plus a **Pattern | Repetitions |
  Time/Cycle | Automation** table, and a friction callout when the session showed
  rage/dead clicks or long context-switch pauses. This is the qualitative spine of
  the report; don't reduce it to a list of links.
- Automation candidate cards, each containing:
  - Signal from data + estimated savings
  - **Current State** (the concrete, painful "before" — what the manual workflow is
    today) **and Future State** (what automation replaces it with). Both are required;
    the Current State is what makes the Future State land.
  - Session evidence block
  - "Watch It Happen" evidence clips — FullStory **deep-link** URLs
    (`…/session/{device-id}:{session-id}:{absolute_ms}`) that jump straight to
    the moment, each with an inline note describing what happens at that timestamp
  - **"How We Got Here" expandable audit trail** (see below)
- **Automation impact summary table** — include the **Min/Event** column
  (Workflow | Level | Monthly Events | Min/Event | Hours Saved | Primary Tool) so the
  `events × minutes` logic stays visible at a glance (full math lives in the audit trail).
- Implementation roadmap (phased)
- Session replay links section
- **Tool-cluster teams only**: an inline caveat — "Groupings for this team are
  inferred from tool-usage patterns; configuring a department/role/team user
  property in FullStory would enable precise, people-grounded numbers."

**Candidate Card Audit Trail** (expandable section per card):

Each automation candidate card includes a collapsible "How We Got Here"
section that makes every number traceable. This is critical for customer
credibility — any number in the deliverable must be auditable back to
its source. The audit trail contains:

1. **Signal Metrics Table**: Each metric that feeds this candidate, with:
   - Signal name
   - Exact MCP query used to build it (verbatim `build_metric` query string)
   - Output type (single_number, top_n)
   - Window (absolute `start_date`–`end_date` — equals the analysis window)
   - Scope: team segment ({property}={value}) or tool-cluster (URL only)
   - Time Property (`focused` / `active` / `N/A`) for time-based metrics
   - Computed value
   - Validation status (Validated / Adjusted / Flagged — from Phase 3.4)
   - If Flagged: explanation of discrepancy

2. **Time-per-Event Justification**:
   - Source: "Session-observed" (with session URL + timestamp) or
     "Default estimate" (with rubric reference)
   - If session-observed: which session, what timestamp range, what was
     happening, how many repetitions were timed
   - The specific value used (e.g., "3 min per copy-paste cycle")

3. **Eliminability Rationale**:
   - The percentage used (e.g., 60%)
   - Why this percentage and not higher/lower
   - Automation level classification justification

4. **Full Calculation**:
   - Show the complete formula with all values plugged in:
     `{volume} events × {time} min × {eliminability}% ÷ 60 = {hours} hrs/month`
   - If multiple signals are summed, show each sub-calculation and the total

**Measurement & Proof Tab** (always present as the last tab):
- Open with the analysis window stamp and a one-line note that every linked
  metric is pinned to that absolute window, so clicking a link shows the same
  number as the report (use the verified URL format from Phase 0.0.3).
- **Baseline Metrics Table**: All key signal metrics with:
  - Metric ID and FullStory link, pinned to the analysis window
    (`https://app.fullstory.com/ui/{org-id}/metrics/create/{metric_id}`, plus
    any date query params confirmed in Phase 0.0.3)
  - Signal name and description
  - Current baseline value (from Phase 1)
  - Suggested dashboard name (`[{prefix}] {Signal Name}`)
  - Which opportunity it maps to (primary signal vs supporting)
  - **How it's calculated** (`{metric_calc}`, shown in an ⓘ hover tooltip):
    compose ONE plain sentence from the ledger row — `Output Type` + `Query` +
    `Window` + `Segment ID` + `Time Property`. No new computation; just restate
    the ledger fields in English. Example: *"Single number — total `Page focused
    time (new)` where URL domain contains `zendesk.com`, scoped to `seg-123`
    (department=Support), Apr 1–30 2026."*
  - **How it's used** (`{metric_role}`, always-visible line under the signal
    name): compose ONE sentence from the ledger `Blueprint Role`. Every metric
    is exactly one of two roles — make the role explicit so the tab is
    unambiguous:
    - **Primary signal** (`role_class=primary`) → name the savings calc it
      feeds, e.g. *"Drives hours-saved for #1 Response Assembly (volume × time ×
      eliminability)."*
    - **Baseline context** (`role_class=context`) → state it's framing only,
      e.g. *"Frames total support time; not used in any savings calc."*
- **Per-Opportunity Success Criteria**: Table with columns for baseline,
  target reduction %, and checkpoints at 30/60/90 days post-implementation
- **Before/After Template**: Pre-filled baseline column with empty
  post-implementation columns for 30/60/90 day results
- **Dashboard Setup Guide**: Instructions for clicking each metric link
  in FullStory → naming it with the suggested name → saving → adding to
  a dashboard
- **Re-Measurement Instructions**: How to run Phase 5 (post-implementation
  measurement) or manually re-compute metrics with shifted date ranges
- **Scoping summary**: list each team's scoping mode (people-scoped with its
  segment definition + member count, or tool-cluster). For people-scoped teams,
  list the segment IDs/links so they can be reused for dashboards and Phase 5.
- If **any** team fell back to tool-cluster, include a note (scoped to those
  teams): "Groupings for {teams} are inferred from tool-usage patterns.
  Configuring a department/role/team user property in FullStory would enable
  precise, people-grounded numbers for them in future analyses." If every team
  was people-scoped, state that team numbers reflect actual member segments.

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
