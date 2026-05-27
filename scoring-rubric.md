# Scoring Rubric

Methodology for identifying, scoring, and ranking automation candidates
from FullStory Workforce data.

---

## Candidate Identification

An automation candidate is any manual workflow where:
1. Quantitative data shows high frequency (100+ events/month)
2. Session evidence confirms it's genuinely manual (not intentional)
3. A realistic automated alternative exists

### Sources of Candidates

| Source | How It Surfaces |
|--------|----------------|
| High copy-paste counts | Manual data transfer between views/tools |
| Repetitive URL path loops | Same navigation sequence repeated 3+ times |
| High report/dashboard views | Data not surfaced where it's needed |
| Manual field population | Form filling that could be auto-populated |
| Cross-tool switching gaps | Long pauses indicating alt-tab for context |
| Queue polling / manual refresh | Lack of auto-refresh or push notifications |
| Duplicate record lookups | Context loss after switching away |
| Low KB/help usage | Tribal knowledge instead of documented answers |
| Auth/login friction | Repeated re-authentication interrupting flow |

---

## Time Estimation Formula

```
Monthly Hours Saved = (Monthly Event Volume × Time per Event × Eliminability %) ÷ 60
```

### Monthly Event Volume

Use the `single_number` metric counts from Phase 1. If multiple signals
contribute to the same workflow, sum them (e.g., "Copied Comment Log" +
"Copied Previous Message" for response assembly).

### Time per Event (Conservative Benchmarks)

Use session observations to calibrate. If no session data is available for
a specific action, use these conservative defaults:

| Action Type | Default Estimate | Notes |
|------------|-----------------|-------|
| Copy-paste cycle | 2–3 min | Copy from source → switch context → paste → edit |
| Form field population | 1–2 min | Per set of fields (not per individual field) |
| Full form submission | 4–5 min | Multi-field form with lookups and validation |
| Cross-tool lookup | 1–3 min | Navigate to other tool, find data, return |
| Record-level research | 5–10 min | Deep dive into a record for context |
| Report-to-record drill-down | 0.5–1.5 min | Per drill-down cycle in a repetitive loop |
| Queue triage (per item) | 1 min | Open → categorize/route → return to queue |
| Manual data entry (per record) | 2–3 min | Creating a new record with manual field entry |
| Search / re-lookup | 1–2 min | Finding something that should have been retained |
| Auth re-login | 1 min | Re-authentication interruption |

**Critical**: Always prefer session-observed times over defaults. If you
watched an agent spend 9 minutes assembling a response via copy-paste,
use that — don't default to 3 minutes.

### Eliminability Percentage

What percentage of the work automation could realistically handle:

| Automation Level | Typical Eliminability | Rationale |
|-----------------|----------------------|-----------|
| Fully Automated | 80–100% | Trigger-based, deterministic logic |
| AI-Assisted | 50–70% | AI handles drafting/heavy lifting, human reviews |
| Process Automated | 25–40% | Reduces steps but doesn't eliminate the task |

Adjust based on specifics:
- If the workflow involves judgment calls → lower eliminability
- If the workflow is purely mechanical → higher eliminability
- If existing integrations/APIs exist → higher eliminability
- If it requires custom development from scratch → lower eliminability

---

## Automation Level Classification

### Fully Automated
Zero human intervention after setup. A trigger fires and the system handles
everything.

**Indicators**:
- The workflow is deterministic (same input → same output)
- No judgment or creativity required
- Existing APIs/integrations can support it
- Error cases are well-defined and handleable

**Examples from past engagements**:
- Auto-populate ticket fields from parsed email body
- Bidirectional CRM-to-sequencing-tool sync on record creation
- Auto-enrich new leads/contacts via ZoomInfo/Clay API
- SSO fix eliminating re-authentication events
- Auto-refresh for queue/list views

### AI-Assisted
AI performs the heavy lifting; a human reviews, edits, and approves.

**Indicators**:
- The workflow requires some judgment but follows patterns
- Quality matters (wrong output has consequences)
- Historical data provides good training signal
- Human review adds meaningful value over full automation

**Examples from past engagements**:
- AI-drafted support responses from ticket context + KB + history
- Call summary generation from Gong recordings → CRM notes
- Smart task prioritization by engagement score and deal value
- AI-powered research surfacing (related tickets, similar issues)

### Process Automated
Workflow is streamlined through data consolidation, smart routing, and
reduced manual steps. The human still does the core work but with less
overhead.

**Indicators**:
- The overhead is in context-gathering, not in the core task
- Data lives in multiple tools and needs consolidation
- Routing/classification logic is straightforward
- Batch operations could replace one-by-one processing

**Examples from past engagements**:
- Unified dashboard replacing cross-tool data consumption
- Embedded analytics on record pages (eliminating report drill-downs)
- Auto-routing deal desk requests by deal size/region
- Batch triage actions (multi-select → bulk categorize)
- Inline editing on list views (replacing open-edit-save-close cycles)

---

## Ranking Methodology

### Primary Sort: Monthly Hours Saved (descending)

The single most important metric. Higher hours = larger business impact.

### Secondary Sort: Implementation Complexity (ascending)

| Complexity | Description | Typical Timeline |
|-----------|-------------|-----------------|
| Low | Configuration change, toggle a setting, enable an existing integration | Days |
| Low-Medium | Configure an existing tool's native integration (e.g., Gong→SFDC sync) | 1–2 weeks |
| Medium | Build a workflow using existing platform capabilities (SFDC Flows, Zendesk triggers) | 2–4 weeks |
| Medium-High | Custom integration or middleware between two tools | 4–8 weeks |
| High | Net-new AI/ML capability or significant platform customization | 8–16 weeks |

### Tertiary Sort: Automation Level

Fully Automated > AI-Assisted > Process Automated

Fully Automated candidates are preferred when hours saved are comparable
because they require zero ongoing human effort after deployment.

---

## Implementation Roadmap Template

Group the ranked candidates into 3 phases:

### Phase 1: Quick Wins (Weeks 1–4)
- Fully Automated items with Low/Low-Medium complexity
- Configuration changes and native integration enablement
- Typically captures 20–30% of total hours saved
- Builds credibility and momentum for later phases

### Phase 2: High Impact (Weeks 5–10)
- AI-Assisted items and Medium complexity integrations
- Largest single-phase hour savings
- Requires some development/integration work
- May involve vendor evaluation or tool procurement

### Phase 3: Foundation (Weeks 11–16)
- Process Automated items requiring workflow redesign
- Cross-tool architecture changes
- Embedded analytics and unified dashboards
- Highest complexity but completes the automation picture

---

## Quality Checks

Before finalizing candidates, verify:

- [ ] Every candidate has both quantitative data AND session evidence
- [ ] Time estimates are conservative (would you defend them to a skeptic?)
- [ ] Eliminability percentages account for edge cases
- [ ] No double-counting: if two candidates share the same event pool,
      split the volume (don't count it twice)
- [ ] Implementation complexity accounts for the org's current tech maturity
- [ ] The total hours saved passes a sanity check against total active time
      (savings shouldn't exceed the team's total working hours)
