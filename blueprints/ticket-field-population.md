# Implementation Blueprint: One-Click Ticket Field Population

**Opportunity**: #3 — Ticket Field Population Ritual
**Platform**: Zendesk
**Automation Level**: Process Automated (reduces 5–6 clicks to 1)
**Estimated Savings**: 19 hours/month (556 events x 2 min each)
**Pattern**: Auto-Population + Batch Processing (hybrid)

---

## 1. Current State

### What the Data Shows

| Signal | Monthly Volume | Source |
|--------|---------------|--------|
| Modified Ticket Category | 195 | FullStory defined event |
| Changed Ticket Field | 361 | FullStory defined event |
| **Total field-setting events** | **556** | Combined |

### What Sessions Revealed

Across 3 observed agent sessions (~7.5 hours, ~37 tickets):

- On nearly every ticket, agents perform an identical **5–6 step ritual**:
  1. Set enterprise/pro dropdown
  2. Set account status to active
  3. Set ticket category
  4. Set follow-up timer (e.g., 5 days)
  5. Set follow-up type (manual)
  6. Choose submit status (Pending, On-Hold, Solved)
- This ritual takes **~30 seconds of mechanical clicking** per ticket
- The sequence is identical across all observed agents — it's an
  unwritten standard operating procedure
- Some agents also manually pause SLA timers, disable CSAT, and take
  ownership — adding 2–3 more clicks to the ritual
- Agents set the same field combinations repeatedly (e.g., "enterprise +
  active + Submit as Pending" is the most common combo)

### The Human Cost

- **19 hours/month** of repetitive clicking on dropdown menus
- Cognitive overhead: agents must remember the 5–6 step sequence
- Inconsistency: some agents skip fields under time pressure, leading
  to incomplete ticket data
- SLA and follow-up timers occasionally missed, causing tickets to fall
  through the cracks
- New agent training burden: memorizing the field ritual takes time

---

## 2. Architecture Overview

### System Diagram

```
┌──────────────────────────────────────────────────────────┐
│                        ZENDESK                           │
│                                                          │
│  SOLUTION A: MACROS (Primary — No Code Required)         │
│  ┌──────────────────────────────────────────────────┐    │
│  │                                                    │   │
│  │  ┌─────────────────┐   ┌──────────────────────┐   │   │
│  │  │  Agent clicks    │──▶│  Macro executes:     │   │   │
│  │  │  single macro    │   │  - Set enterprise    │   │   │
│  │  │  button          │   │  - Set active        │   │   │
│  │  │                  │   │  - Set category      │   │   │
│  │  │  e.g., "Resolve  │   │  - Set follow-up 5d  │   │   │
│  │  │   Enterprise     │   │  - Set follow-up     │   │   │
│  │  │   Ticket"        │   │    type: manual      │   │   │
│  │  │                  │   │  - Submit as Pending  │   │   │
│  │  └─────────────────┘   └──────────────────────┘   │   │
│  │                                                    │   │
│  │  Old: 6 clicks across 6 dropdowns (~30 sec)        │   │
│  │  New: 1 click on macro button (~2 sec)             │   │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  SOLUTION B: TRIGGERS (Secondary — Auto-Set on Intake)   │
│  ┌──────────────────────────────────────────────────┐    │
│  │                                                    │   │
│  │  When ticket is created with tag "auto_populated": │   │
│  │  - IF Org has plan = enterprise:                   │   │
│  │    → Auto-set enterprise, active                   │   │
│  │  - IF ticket category detectable from subject:     │   │
│  │    → Auto-set category                             │   │
│  │                                                    │   │
│  │  Pairs with Opportunity #2 (auto-population)       │   │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  SOLUTION C: CONDITIONAL FIELDS (Longer-term)            │
│  ┌──────────────────────────────────────────────────┐    │
│  │                                                    │   │
│  │  Zendesk Conditional Fields configuration:         │   │
│  │  - When plan type = enterprise:                    │   │
│  │    → Auto-show relevant fields                     │   │
│  │    → Pre-select common values                      │   │
│  │    → Hide irrelevant options                       │   │
│  │                                                    │   │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

### Approach Comparison

This opportunity is best solved with a **layered approach** — macros for
immediate wins, triggers for deeper automation, and conditional fields for
long-term cleanup:

| Approach | Impact | Effort | Best For |
|----------|--------|--------|----------|
| **Macros** (recommended first) | High — 1 click replaces 6 | Very low — config only | Immediate deployment |
| **Triggers** (pair with Opp #2) | Medium — auto-sets some fields | Low — Zendesk config | Fields derivable from ticket data |
| **Conditional Fields** | Medium — simplifies agent UI | Medium — requires field audit | Reducing dropdown clutter |
| **Zendesk Apps Framework sidebar** | High — custom UI | High — custom development | Complex conditional logic |

**Recommendation**: Start with macros (deploy in hours, not days). Layer
triggers on top when Opportunity #2 is live. Add conditional fields during
a quarterly Zendesk cleanup sprint.

---

## 3. Detailed Build Steps

### Prerequisites

- [ ] Zendesk Admin access (to create macros, modify triggers)
- [ ] List of all ticket field names, IDs, and valid dropdown values
- [ ] Understanding of the most common field combinations agents use
      (audit 20–30 recent tickets to identify top patterns)

### Step 1: Audit Current Field Patterns

Before creating macros, identify the most common field combinations:

1. Export recent tickets (Admin > Explore or API:
   `GET /api/v2/search.json?query=type:ticket created>2024-01-01`)
2. Analyze the top 5 combinations of:
   - Plan type (enterprise, pro, free)
   - Account status (active, trial, churned)
   - Submit action (Pending, On-Hold, Solved, New)
   - Follow-up timer value (1 day, 3 days, 5 days, none)
3. Document the top combinations — these become your macros

**Expected top patterns** (based on session observations):

| Pattern | Frequency | Fields Set |
|---------|-----------|------------|
| Enterprise Active → Pending 5-day | ~40% | enterprise, active, pending, 5-day, manual |
| Enterprise Active → Solved | ~25% | enterprise, active, solved, disable CSAT |
| Enterprise Active → On-Hold | ~15% | enterprise, active, on-hold, pause SLA |
| Pro Active → Pending 3-day | ~10% | pro, active, pending, 3-day, manual |
| Other combinations | ~10% | Various |

### Step 2: Create Macros

Go to **Admin Center > Workspaces > Agent Tools > Macros > Add Macro**.

**Macro 1: "Enterprise → Pending (5-day follow-up)"**
- Actions:
  - Set Plan Type: `enterprise`
  - Set Account Status: `active`
  - Set Follow-up Timer: `5 days`
  - Set Follow-up Type: `manual`
  - Set Ticket Status: `pending`
- Description: "Sets enterprise/active fields and submits as pending
  with 5-day follow-up"

**Macro 2: "Enterprise → Solved"**
- Actions:
  - Set Plan Type: `enterprise`
  - Set Account Status: `active`
  - Set CSAT: `disabled` (if applicable)
  - Set Ticket Status: `solved`
- Description: "Sets enterprise/active fields and resolves ticket"

**Macro 3: "Enterprise → On-Hold (SLA Paused)"**
- Actions:
  - Set Plan Type: `enterprise`
  - Set Account Status: `active`
  - Set SLA Pause: `true`
  - Set Ticket Status: `on-hold`
- Description: "Sets enterprise/active fields, pauses SLA, submits
  as on-hold"

**Macro 4: "Pro → Pending (3-day follow-up)"**
- Actions:
  - Set Plan Type: `pro`
  - Set Account Status: `active`
  - Set Follow-up Timer: `3 days`
  - Set Follow-up Type: `manual`
  - Set Ticket Status: `pending`

**Naming convention**: `{Plan} → {Action} ({detail})` — agents can
quickly find the right macro by plan type.

### Step 3: Organize Macros for Quick Access

1. **Macro categories**: Group the new macros under a category like
   "Quick Submit" so they appear together in the macro dropdown
2. **Keyboard shortcut**: Train agents to use the macro shortcut
   (type `/` or use the macro button in the reply toolbar)
3. **Pin top macros**: If Zendesk supports pinning, pin the top 2–3
   most-used macros for one-click access

### Step 4: Create Auto-Set Triggers (pairs with Opportunity #2)

When Opportunity #2 is deployed, add triggers that pre-set fields based
on auto-populated data:

**Trigger: "Auto-Set Enterprise Fields"**
- Conditions (ALL):
  - Tag contains: `auto_populated`
  - Plan Type field value: `enterprise`
- Actions:
  - Set Account Status: `active`
  - Add tag: `fields_auto_set`

**Trigger: "Auto-Categorize from Subject"**
- Conditions (ALL):
  - Ticket is Created
- Actions:
  - Use Liquid markup to parse subject line:
    - If subject contains "quota" → Set Category: `quota`
    - If subject contains "billing" → Set Category: `billing`
    - If subject contains "SSO" or "login" → Set Category: `authentication`

### Step 5: Configure Conditional Fields (longer-term)

1. Go to **Admin Center > Objects and Rules > Tickets > Conditional Fields**
2. Create conditions:
   - When Plan Type = `enterprise`:
     - Show: SLA Pause option, Enterprise-specific categories
     - Hide: Free-tier categories, upgrade prompts
   - When Plan Type = `pro`:
     - Show: Pro-specific categories
     - Hide: Enterprise-only options
3. Set default values where safe:
   - Account Status defaults to `active` (most common)
   - Follow-up Type defaults to `manual` (most common)

### Step 6: Handle Edge Cases

| Scenario | Behavior |
|----------|----------|
| Agent needs a non-standard combo | Manually set fields (macros don't prevent manual editing) |
| New plan type added | Create new macro, update trigger conditions |
| Macro conflicts with trigger | Trigger runs first (on creation), macro overrides if agent clicks it |
| Agent applies wrong macro | Agent edits fields manually — macros are non-destructive |

---

## 4. Monitoring & Quality Tracking

**In FullStory (post-deployment):**
- "Modified Ticket Category" events (should decrease from 195/month)
- "Changed Ticket Field" events (should decrease from 361/month)
- Time spent on field dropdowns per ticket (should approach zero for
  macro-using agents)
- Macro button click events (new signal — shows adoption)

**In Zendesk:**
- Macro usage report (Admin > Explore):
  - Which macros are used most often
  - Which agents use macros vs manual field setting
  - Macro usage trend over time (should increase week over week)
- Ticket data completeness:
  - % of tickets with all required fields populated
  - Should increase as macros enforce consistency

---

## 5. Validation & Rollout

### Testing Checklist

- [ ] Each macro sets all expected fields correctly
- [ ] Macro does not overwrite fields that were already set
- [ ] Macro works from both the reply toolbar and keyboard shortcut
- [ ] Trigger auto-sets fields when paired with auto_populated tag
- [ ] Conditional fields show/hide correctly based on plan type
- [ ] Subject-line categorization trigger fires accurately
- [ ] No conflicts between macros and triggers
- [ ] Edge case: agent can override macro-set fields manually

### Phased Rollout

**Week 1: Macro Launch**
- Create top 4 macros (covers ~90% of field combinations)
- Announce to support team with 2-minute training video/doc
- Track: macro usage rate, agent feedback

**Week 2-3: Trigger Layer**
- Deploy auto-set triggers (requires Opportunity #2 to be live)
- Monitor: trigger fire rate, field accuracy
- Tune: subject-line categorization rules based on misclassifications

**Week 4+: Conditional Fields + Refinement**
- Configure conditional fields to simplify the dropdown UI
- Add/remove macros based on actual usage data
- Target: > 80% of tickets resolved using macros

### Rollback Plan

1. **Macros**: Simply delete them — agents return to manual field setting.
   Zero risk, zero data impact.
2. **Triggers**: Disable the trigger (Admin > Triggers > disable).
   Already-set fields remain, no new tickets affected.
3. **Conditional fields**: Revert to showing all fields (Admin >
   Conditional Fields > delete rules).

### Success Criteria

| Metric | Baseline | Target (90 days) |
|--------|----------|-------------------|
| Manual field-change events/month | 556 | < 150 (73% reduction) |
| Time per ticket on field setting | ~30 sec (6 clicks) | ~2 sec (1 macro click) |
| Macro adoption rate | 0% | > 80% of tickets |
| Ticket data completeness | Current baseline | > 95% fields populated |
| Follow-up timers missed | Current baseline | 50% reduction |

---

## 6. Cost Estimate

| Component | Estimated Monthly Cost |
|-----------|----------------------|
| Zendesk macros (native feature) | $0 |
| Zendesk triggers (native feature) | $0 |
| Conditional fields (native feature) | $0 |
| **Total** | **$0/month** |

This is the rare automation that costs literally nothing — it's pure
Zendesk configuration. The 19 hours/month savings is essentially free.

---

## 7. Adaptations for Other Orgs

| Platform | Macro Equivalent | Trigger Equivalent | Conditional Fields |
|----------|-----------------|-------------------|-------------------|
| Zendesk | Macros | Triggers + Automations | Conditional Fields |
| Freshdesk | Canned Actions / Scenarios | Automations | Dynamic Ticket Forms |
| Intercom | Macros (saved replies + actions) | Workflows | Custom Bot flows |
| ServiceNow | UI Actions / Client Scripts | Business Rules | UI Policies |
| Salesforce Service Cloud | Quick Actions / Macros | Process Builder / Flow | Page Layouts + Record Types |

The macro approach is universally applicable — every support platform has
some form of "bundle multiple actions into one click."
