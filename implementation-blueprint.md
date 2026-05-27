---
name: implementation-blueprint
description: >-
  Generate a detailed, step-by-step implementation blueprint for any automation
  opportunity identified in a workforce automation analysis. Takes a ranked
  opportunity (from the workforce analysis deliverable) and produces a complete
  build guide covering current state, architecture, build steps, validation,
  and rollout. Use when a user selects a specific automation opportunity and
  asks "how do we actually build this?"
---

# Automation Implementation Blueprint Generator

Produces a complete, team-ready implementation guide for a single automation
opportunity. The output is a document an engineering or ops team can pick up
and start building from — no further discovery required.

## When to Use This Skill

- User selects an opportunity from the ranked automation table and wants a
  detailed build plan
- User says "how do we automate this?" or "build the implementation plan for
  opportunity #N"
- User wants to hand off an automation project to their engineering team

## Inputs Required

Before starting, confirm you have:

1. **The opportunity details** — from the workforce analysis deliverable:
   - Name and rank
   - Signal data (event names, monthly volumes)
   - Time estimate (events x time-per-event = hours/month)
   - Automation level (Fully Automated, AI-Assisted, Process Automated)
   - Session evidence (behavioral patterns observed)
   - Session URLs with timestamps

2. **The customer's tool stack** — which platform the opportunity lives in
   (e.g., Zendesk, Salesforce, Outreach, Gong). Confirm with the user if
   unclear.

3. **Any constraints** — budget, timeline, security requirements, preferred
   vendors, existing integrations.

If any of these are missing, ask the user before proceeding.

## Blueprint Structure

Every blueprint follows 6 sections. Each section has specific requirements
detailed below.

---

### Section 1: Current State

**Purpose**: Ground the blueprint in evidence. Anyone reading should
immediately understand what's broken and why it matters.

**Must include**:

| Element | Source | Example |
|---------|--------|---------|
| Signal data table | Workforce analysis metrics | "Copied Comment Log: 2,487/month" |
| Session observations | Session deep dives | "Agent copied from history 5+ times per ticket" |
| Human cost summary | Time estimate + qualitative impact | "255 hrs/month, inconsistent quality, slow onboarding" |

**Template**:

```markdown
## 1. Current State

### What the Data Shows

| Signal | Monthly Volume | Source |
|--------|---------------|--------|
| {event_name_1} | {count_1} | FullStory defined event |
| {event_name_2} | {count_2} | FullStory defined event |
| **Total** | **{sum}** | Combined |

### What Sessions Revealed

Across {N} observed sessions (~{hours} hours, ~{items} {unit}):

- {Key behavioral pattern 1}
- {Key behavioral pattern 2}
- {Key behavioral pattern 3}
- {Surprising or notable finding}

### The Human Cost

- **{hours}/month** of {role} time spent on this workflow
- {Quality/consistency impact}
- {Speed impact}
- {Onboarding/scaling impact}
- {Systemic issue — e.g., tribal knowledge, missing tooling}
```

---

### Section 2: Architecture Overview

**Purpose**: Show the team what they're building before diving into steps.
A senior engineer should be able to look at this section and estimate LOE.

**Must include**:

1. **System diagram** (ASCII art) — what components connect to what,
   directional data flow with numbered steps
2. **Data flow narrative** — 4-6 numbered steps from trigger to outcome
3. **Build vs Buy decision matrix** — at least 3 options compared

**Guidelines for the system diagram**:
- Use ASCII box-and-arrow diagrams (universally renderable)
- Show: trigger source → middleware/logic → external APIs → output destination
- Label each arrow with what data flows through it
- Include the human touchpoint (where the person reviews/approves)

**Build vs Buy matrix template**:

```markdown
| Approach | Pros | Cons |
|----------|------|------|
| **Native platform feature** | {pros} | {cons} |
| **Custom middleware** (recommended) | {pros} | {cons} |
| **Third-party vendor** ({name}) | {pros} | {cons} |
```

Always include a recommendation with rationale.

---

### Section 3: Detailed Build Steps

**Purpose**: A numbered sequence the team follows. Each step should be
completable by one person in one sitting.

**Must include**:

1. **Prerequisites checklist** — accounts, API keys, permissions needed
   before starting anything
2. **Numbered steps** — each with:
   - What to do (specific UI paths, API endpoints, config values)
   - Code/config examples (pseudocode or real code, clearly labeled)
   - Expected result after completing the step
3. **Platform-specific details** — exact menu paths, field names, API
   routes (not vague descriptions like "set up the integration")

**Step detail calibration by automation pattern**:

| Pattern | Key Build Steps |
|---------|----------------|
| **AI Drafting** | API token setup → Middleware function → Context fetching → LLM prompt template → Draft write-back → Agent workflow macros → Monitoring |
| **Data Sync** | Source API auth → Destination API auth → Field mapping → Sync trigger (webhook/polling) → Conflict resolution logic → Error handling → Dedup strategy |
| **Auto-Population** | Trigger definition → Input parsing logic → Field mapping rules → Write-back API calls → Validation rules → Fallback behavior |
| **Data Consolidation** | Data source inventory → API/embed auth → Layout/dashboard design → Data refresh strategy → Access control |
| **Workflow Routing** | Classification rules or AI model → Routing logic → Assignment/queue configuration → Escalation paths → Override mechanism |
| **Batch Processing** | Selection criteria → Batch action definition → Preview/confirmation UX → Execution logic → Rollback/undo capability |

**For AI-Assisted patterns, always include**:
- A complete LLM prompt template with placeholders
- Token/cost estimates based on expected call volume
- Instructions for prompt iteration based on agent feedback

**For Fully Automated patterns, always include**:
- The exact trigger conditions (when does it fire?)
- The exact output (what gets written/created/updated?)
- Error handling for every known failure mode

---

### Section 4: Monitoring & Quality Tracking

**Purpose**: How does the team know it's working? Define metrics before
launch, not after.

**Must include**:

| Metric Category | What to Track | Where |
|----------------|---------------|-------|
| **FullStory (post-deploy)** | Reduction in the original signal events | FullStory metrics |
| **Middleware/system logs** | Execution count, latency, error rate | Application logs |
| **Platform-specific** | Acceptance rate, edit distance, user satisfaction | The platform itself |

**Template**:

```markdown
### In FullStory (post-deployment)
- {Original signal event} count (should decrease from {baseline}/month)
- {Behavioral metric} (should {improve/decrease})
- {Time metric} (should decrease)

### In Middleware/System Logs
- Execution count per day
- Latency (p50, p95)
- Error rate
- {Resource usage metric — tokens, API calls, etc.}

### In {Platform Name}
- {Acceptance/usage rate of the automation}
- {Quality metric — edit distance, override rate, etc.}
- {End-user impact metric — resolution time, satisfaction, etc.}
```

---

### Section 5: Validation & Rollout

**Purpose**: De-risk deployment. No big-bang launches.

**Must include**:

1. **Testing checklist** — 8-12 specific test cases (not generic "test it works")
2. **Phased rollout** — always 3 phases:
   - **Shadow/Pilot** (Week 1-2): Small group, optional use, gather feedback
   - **Expanded Pilot** (Week 3-4): Full team, still optional, track metrics
   - **Default Workflow** (Week 5+): Standard process, measure vs baseline
3. **Rollback plan** — how to disable in < 5 minutes if something breaks
4. **Success criteria table** — baseline vs target for 3-5 metrics

**Success criteria template**:

```markdown
| Metric | Baseline | Target (90 days) |
|--------|----------|-------------------|
| {Primary signal event}/month | {current} | < {target} ({X}% reduction) |
| {Time metric} | {current} | {target} |
| {Automation-specific metric} | N/A | > {target} |
| {Quality metric} | {current} | No decrease |
```

---

### Section 6: Cost Estimate & Platform Adaptations

**Purpose**: Answer "what will this cost?" and "what if we use a different
tool?"

**Must include**:

1. **Monthly cost table** — itemized by component
2. **ROI framing** — cost vs hours saved
3. **Platform adaptation table** — how the same pattern applies to 3-4
   alternative platforms (for orgs considering tool migration or with
   multiple platform instances)

**Platform adaptation template**:

```markdown
| Platform | Trigger Mechanism | API for Context | Output Delivery |
|----------|------------------|-----------------|-----------------|
| {Primary platform} | {mechanism} | {API} | {delivery method} |
| {Alt platform 1} | {mechanism} | {API} | {delivery method} |
| {Alt platform 2} | {mechanism} | {API} | {delivery method} |
```

---

## Automation Pattern Reference

When building a blueprint, first classify the opportunity into one of these
patterns. The pattern determines which build-step template to follow and
which architectural components are required.

### AI Drafting

**What it automates**: A human currently assembles content manually
(copying, researching, composing) — AI generates a draft for review.

**Architecture skeleton**:
```
Trigger (new item/event) → Context Assembly (fetch relevant data) →
LLM Call (generate draft) → Draft Delivery (write back to platform) →
Human Review → Send/Approve
```

**Key decisions**:
- Which LLM provider? (cost vs quality vs latency tradeoffs)
- What context to include? (more context = better drafts = higher cost)
- Where does the draft appear? (internal note, sidebar, draft field)
- How does the human approve? (one-click vs edit-and-send)

**Examples**: Response drafting, call summary generation, meeting notes,
email composition, report narrative generation.

### Data Sync

**What it automates**: A human manually copies data from Tool A to Tool B
— a bidirectional or unidirectional sync keeps records aligned.

**Architecture skeleton**:
```
Trigger (record created/updated in Tool A) → Webhook/Polling →
Middleware (field mapping + transformation) →
API Write to Tool B → Confirmation/logging
```

**Key decisions**:
- Unidirectional or bidirectional?
- Conflict resolution strategy (last-write-wins, source-of-truth, merge)
- Deduplication logic (match on email? external ID? name fuzzy match?)
- Sync frequency (real-time webhook vs scheduled batch)

**Examples**: CRM↔sequencing tool sync, prospect creation across tools,
contact enrichment data push, deal stage sync.

### Auto-Population

**What it automates**: A human reads input data and manually fills form
fields — parsing logic does it automatically.

**Architecture skeleton**:
```
Trigger (new record/ticket/form) → Parse Input (regex, NLP, API lookup) →
Field Mapping → API Write (populate fields) → Validation
```

**Key decisions**:
- Parse method: regex (structured input) vs NLP (unstructured) vs
  API lookup (external data source)?
- What if parsing fails? (leave blank, flag for human, best-guess + flag)
- Which fields are safe to auto-populate vs require human confirmation?

**Examples**: Ticket field population, org/account ID lookup, lead
source attribution, form pre-fill from CRM data.

### Data Consolidation

**What it automates**: A human checks 3-5 tools for context — a unified
view brings everything into one screen.

**Architecture skeleton**:
```
Multiple Data Sources → API/Embed Integration →
Unified View (dashboard, sidebar, embedded panel) →
Auto-Refresh → Access Control
```

**Key decisions**:
- Embedded widget vs separate dashboard vs sidebar app?
- Real-time API calls vs cached/synced data?
- What's the refresh strategy?
- How to handle access control across data sources?

**Examples**: Customer 360 view, deal room, support ticket context panel,
account health dashboard, cross-tool reporting.

### Workflow Routing

**What it automates**: A human manually classifies items and routes them
to the right queue/person — rules or AI handle classification and assignment.

**Architecture skeleton**:
```
Trigger (new item) → Classification (rules/AI) →
Routing Logic (assignment rules, round-robin, load-balance) →
Assign + Notify → Escalation Path
```

**Key decisions**:
- Rules-based (deterministic) vs AI-based (probabilistic)?
- How to handle edge cases / low-confidence classifications?
- Override mechanism for manual rerouting?
- Escalation criteria and paths?

**Examples**: Ticket triage, deal desk routing, lead assignment, support
tier escalation, approval routing.

### Batch Processing

**What it automates**: A human performs the same action one-by-one on
multiple items — batch operations handle many at once.

**Architecture skeleton**:
```
Selection (filter/multi-select) → Preview (show what will change) →
Batch Execute → Result Summary → Undo/Rollback
```

**Key decisions**:
- How does the user select items? (filter criteria vs manual selection)
- Preview before execution? (critical for destructive actions)
- Async execution for large batches?
- Undo/rollback capability?

**Examples**: Bulk ticket triage, pipeline cleanup, batch lead assignment,
mass email operations, bulk record updates.

---

## Process

1. **Receive the opportunity** — get the name, rank, signal data, time
   estimate, automation level, and session evidence from the workforce
   analysis deliverable.

2. **Classify by pattern** — match the opportunity to one of the 6
   automation patterns above. If it spans multiple patterns, pick the
   dominant one and note secondary elements.

3. **Identify the platform** — confirm which tool the workflow lives in
   (Zendesk, Salesforce, etc.). Look up that platform's:
   - Webhook/trigger capabilities
   - REST API documentation
   - Native automation features (Flows, Triggers, Automations)
   - Marketplace apps/integrations that could help

4. **Write Section 1 (Current State)** — pull directly from the workforce
   analysis data. Copy the numbers and session evidence verbatim. Add
   the human cost framing.

5. **Write Section 2 (Architecture)** — follow the pattern skeleton.
   Adapt the system diagram to the specific platform's API and integration
   points. Include the build-vs-buy matrix.

6. **Write Section 3 (Build Steps)** — follow the pattern-specific build
   step template. Include real API endpoints, menu paths, config values,
   and code examples. Every step should be actionable without needing to
   look up documentation.

7. **Write Section 4 (Monitoring)** — define metrics across FullStory,
   middleware, and the platform. Tie back to the original signal events
   from the workforce analysis.

8. **Write Section 5 (Validation & Rollout)** — write specific test cases
   (not generic ones), define the 3-phase rollout, include rollback
   instructions, and set success criteria with baseline numbers from the
   workforce analysis.

9. **Write Section 6 (Cost & Adaptations)** — estimate costs based on
   expected call volume, include platform alternatives.

10. **Save the blueprint** — write to
    `~/.cursor/skills/workforce-automation-analysis/blueprints/{slug}.md`
    where `{slug}` is a kebab-case name derived from the opportunity name.

## Output Format

The blueprint is a single Markdown file. Use the reference implementation
at `blueprints/zendesk-response-assembly.md` as the gold standard for
tone, depth, and structure.

**Calibration guidelines**:
- Build steps should be specific enough that a mid-level engineer can
  follow them without asking questions
- Code examples can be pseudocode but must show real API endpoints and
  data structures
- Time estimates should be conservative (defensible to a skeptic)
- The blueprint should be 300-500 lines of Markdown
- Use tables for structured data, not prose
- Include checkboxes for prerequisites and testing (actionable checklists)

## Reference Implementation

See `blueprints/zendesk-response-assembly.md` for a complete example
covering:
- AI Drafting pattern
- Zendesk platform
- Response Assembly opportunity (#1, 255 hrs/month)

This reference demonstrates the target quality, depth, and format
for all blueprints.
