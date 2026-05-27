# FullStory Workforce Automation Analysis — Gemini Playbook

You are running a structured automation discovery analysis using FullStory
Workforce data. Your goal is to identify manual workflows, quantify time
waste with data, gather session replay evidence, and produce a polished
HTML deliverable with ranked automation candidates and implementation roadmaps.

Follow the phases below in order. Do not skip phases.

All FullStory MCP tools use the prefix `mcp_fullstory_`. For example:
`mcp_fullstory_build_metric`, `mcp_fullstory_compute_metric`, etc.

---

## Phase 0: Orientation

Build a data-driven inventory of the org's tool stack.

### Step 0.1: Discover All Tools

Do NOT assume which tools are in use. Query the org's actual data:

```
mcp_fullstory_build_metric(
  query="page views grouped by top-level URL domain",
  output_type="top_n",
  time_range="last_30_days"
)
```

Then compute it:

```
mcp_fullstory_compute_metric(metric_id=<returned_id>)
```

This returns every domain with captured activity, ranked by page view volume.

### Step 0.2: Get Active Time Per Tool

For each significant domain discovered (skip auth/SSO, CDNs, etc.):

```
mcp_fullstory_build_metric(
  query="total active time where URL domain contains {domain}",
  output_type="single_number",
  time_range="last_30_days"
)
→ mcp_fullstory_compute_metric(metric_id=<returned_id>)
```

### Step 0.3: Present and Scope

Present the ranked tool inventory to the user (domain, page views,
active time). Recommend team groupings based on tool clusters:

| Tool Cluster | Suggested Team |
|-------------|---------------|
| Zendesk, Intercom, Freshdesk, ServiceNow | Customer Support |
| SFDC + Outreach/Salesloft + Gong/Chorus + ZoomInfo | Sales Operations |
| SFDC + Looker/Tableau + Gong + CPQ | RevOps / Deal Desk |
| Jira, Linear, Confluence, GitHub | Engineering Ops |
| HubSpot, Marketo, LinkedIn, Drift | Marketing Ops |

Ask the user which teams to analyze (recommend 2–3). Confirm time range
(default: last 30 days) and sessions per team (default: 3).

---

## Phase 1: Quantitative Discovery (per team)

Run for each team the user selected.

### Step 1.1: URL Path Breakdown

For each tool domain in the team's stack:

```
mcp_fullstory_build_metric(
  query="page views grouped by URL path where URL domain contains {domain}",
  output_type="top_n",
  time_range="last_30_days"
)
→ mcp_fullstory_compute_metric(metric_id=<returned_id>)
```

Also get unique users:

```
mcp_fullstory_build_metric(
  query="unique users where URL domain contains {domain}",
  output_type="single_number",
  time_range="last_30_days"
)
→ mcp_fullstory_compute_metric(metric_id=<returned_id>)
```

### Step 1.2: Discover Interaction Signals

Search for org-specific instrumentation using the actual tool names
discovered in Phase 0:

```
mcp_fullstory_discover_org_context(
  queries=["{tool_name}", "copy", "paste", "submit", "click",
           "field", "search", "ticket", "task", "deal"]
)
```

For each relevant defined event or named element found, count occurrences:

```
mcp_fullstory_build_metric(
  query="count of {defined_event_name} events",
  output_type="single_number",
  time_range="last_30_days"
)
→ mcp_fullstory_compute_metric(metric_id=<returned_id>)
```

### Step 1.3: Analyze Patterns

From the URL path breakdown, identify:
- **Data consumption**: report/dashboard paths with high view counts
- **Queue/list views**: ticket queues, task lists with high counts
- **Record-level views**: individual records viewed repeatedly
- **Search frequency**: search paths indicating context loss
- **Low-usage areas**: KB, docs, help pages (low usage = missed opportunity)

### Step 1.4: Document Findings

For each tool, record:
- Tool name and domain
- Page views, active time, unique users
- Top URL paths with counts
- Notable defined events and their counts
- Any anomalies (e.g., very high report consumption, low KB usage)

---

## Phase 2: Session Deep Dives (per team)

### Step 2.1: Find Sessions

For each team, use `get_sessions` with a metric targeting the primary tool:

```
mcp_fullstory_get_sessions(metric_id=<team_tool_metric_id>, limit=10)
```

Select 3 sessions with:
- High page counts (sustained work, not drive-by)
- Multiple domains if possible (cross-tool switching)
- Friction signals (rage clicks, dead clicks)

### Step 2.2: Analyze Each Session

**Important**: Analyze one session at a time to manage context. For each:

```
mcp_fullstory_get_session_events(
  device_id=<device_id>,
  session_id=<session_id>
)
```

From the event transcript, extract:
1. **Workflow patterns**: What did the user do? How many times did they
   repeat the same sequence?
2. **Timing**: How long did each cycle take?
3. **Friction**: Rage clicks, dead clicks, long pauses (30s+ = alt-tab
   to another app), mouse thrash
4. **Manual data transfer**: Copy-paste chains between views/tools
5. **Navigation inefficiency**: Drill-downs requiring 3+ page loads
6. **Duplicate lookups**: Same record searched/opened more than once

**After analyzing each session, summarize the key findings in a compact
format before moving to the next session.** This preserves context window
space. Discard the raw event transcript after summarizing.

### Step 2.3: Record Evidence Clips

For each session, identify the 2–3 most compelling moments showing manual
work. Record:
- Session URL: `https://app.fullstory.com/ui/{org-id}/session/{device-id}:{session-id}`
- Timestamp (minutes:seconds from session start)
- Description of what's happening
- Which automation candidate it supports

---

## Phase 3: Synthesis & Scoring

### Step 3.1: Identify Automation Candidates

For each manual workflow observed, create a candidate:

| Field | How to Determine |
|-------|-----------------|
| Name | Descriptive workflow name |
| Signal from Data | Which Phase 1 metrics support this |
| Session Evidence | Timestamps from Phase 2 |
| Monthly Event Volume | Phase 1 metric counts |
| Time per Event | Session observation (or conservative default below) |
| Eliminability % | See classification below |
| Monthly Hours Saved | volume × time × eliminability ÷ 60 |

**Time-per-event defaults** (use session observations when available):

| Action | Default |
|--------|---------|
| Copy-paste cycle | 2–3 min |
| Form field population | 1–2 min |
| Full form submission | 4–5 min |
| Cross-tool lookup | 1–3 min |
| Record-level research | 5–10 min |
| Report drill-down cycle | 0.5–1.5 min |
| Queue triage per item | 1 min |
| Manual record creation | 2–3 min |
| Search / re-lookup | 1–2 min |
| Auth re-login | 1 min |

Target 5–6 candidates per team.

### Step 3.2: Classify Automation Level

- **Fully Automated** (80–100% eliminable): Zero human intervention.
  Examples: auto-enrichment, bidirectional sync, auto-populate fields.

- **AI-Assisted** (50–70% eliminable): AI drafts, human reviews.
  Examples: AI-drafted responses, call summaries, smart prioritization.

- **Process Automated** (25–40% eliminable): Fewer manual steps.
  Examples: unified dashboards, batch actions, auto-routing.

### Step 3.3: Rank All Candidates

Sort by: monthly hours saved (descending), then implementation complexity
(ascending), then automation level (Fully Automated first).

### Step 3.4: Build Roadmaps (per team)

- **Phase 1 — Quick Wins**: Fully Automated + Low complexity
- **Phase 2 — High Impact**: AI-Assisted + Medium complexity
- **Phase 3 — Foundation**: Process Automated + Higher complexity

---

## Phase 4: Generate HTML Deliverable

Create a single self-contained HTML file with tab-based navigation.
No external dependencies. All CSS inline, all JS inline.

### File Structure

```html
<div class="tab-bar">
  <button class="tab active" data-tab="overview">Overview</button>
  <!-- One tab per team -->
</div>

<div id="section-overview" class="section">
  <!-- Hero stats, team cards, ranked table, key moments -->
</div>

<!-- One section per team -->
<div id="section-{team_id}" class="section" style="display:none">
  <!-- Team blueprint content -->
</div>

<script>
function switchTab(tab, anchor) {
  document.querySelectorAll('.section').forEach(s => s.style.display = 'none');
  document.getElementById('section-' + tab).style.display = 'block';
  document.querySelectorAll('.tab').forEach(t => t.classList.remove('active'));
  document.querySelector('.tab[data-tab="' + tab + '"]').classList.add('active');
  if (anchor) {
    setTimeout(() => {
      var el = document.getElementById(anchor);
      if (el) el.scrollIntoView({ behavior: 'smooth', block: 'start' });
    }, 100);
  } else {
    window.scrollTo(0, 0);
  }
}
</script>
```

### Overview Tab Must Include

1. **Hero section**: total hours saved, FTE equivalent, workflow count,
   session count — dark gradient background (#1a1a2e → #6c5ce7)
2. **Team cards**: one card per team linking to its tab via switchTab()
3. **Summary bar**: counts by automation level (Fully/AI/Process)
4. **Ranked table**: ALL candidates across all teams — rank, team pill,
   workflow name, automation level pill, hours saved, tools, view link
5. **Key Moments**: 8–10 curated session replay links with timestamps

### Per-Team Blueprint Tabs Must Include

1. **Executive callout**: key numbers in a blue info callout box
2. **Tool volume breakdown**: stat cards for page views, active time, users
3. **URL path analysis**: horizontal bar charts showing where time goes
4. **Behavioral signals**: copy-paste counts, field changes, etc.
5. **Automation candidate cards** (each with id for anchor linking):
   - Card header: #{rank}: {name} + automation level pill
   - Signal from data + estimated savings
   - Future state description
   - Session evidence block
   - "Watch It Happen" evidence clips with timestamped FullStory URLs
6. **Impact summary table**: all candidates with hours saved
7. **Implementation roadmap**: 3 phased cards (Quick Win, High Impact, Foundation)
8. **Session replay links**: full session URLs

### Design Tokens

```css
:root {
  --accent: #6c5ce7;        /* FullStory purple */
  --success: #10b981;       /* Fully Automated */
  --info: #3b82f6;          /* AI-Assisted */
  --warning: #f59e0b;       /* Process Automated */
  --text: #1a1a2e;
  --text-secondary: #64748b;
  --border: #e2e8f0;
  --bg-alt: #f7f8fa;
}
```

Use SVG for charts (no external libraries). Make it responsive and
print-friendly. All FullStory session links use `target="_blank"`.

---

## Quality Checklist

Before delivering the report:

- [ ] Every automation candidate has BOTH quantitative data AND session evidence
- [ ] Time estimates are conservative (defensible to a skeptic)
- [ ] No double-counting of events across candidates
- [ ] Total hours saved passes sanity check vs total active time
- [ ] All session replay links are valid URLs with correct org-id
- [ ] HTML file opens correctly in a browser with no external dependencies
- [ ] Tab navigation works (switchTab function, anchor scrolling)
- [ ] All evidence clips have specific timestamps
- [ ] Ranked table links correctly navigate to candidate cards
