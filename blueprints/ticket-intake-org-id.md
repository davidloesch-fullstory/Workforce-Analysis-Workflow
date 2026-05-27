# Implementation Blueprint: Auto-Populated Ticket Intake & Org ID

**Opportunity**: #2 — New Ticket Intake & Org ID Entry
**Platform**: Zendesk
**Automation Level**: Fully Automated (trigger-based, zero human intervention)
**Estimated Savings**: 55 hours/month (830 events x 4 min each)
**Pattern**: Auto-Population

---

## 1. Current State

### What the Data Shows

| Signal | Monthly Volume | Source |
|--------|---------------|--------|
| Input Org ID | 830 | FullStory defined event |
| Requester email copy-paste from ticket body | Included in above | Session observation |
| **Total manual intake events** | **830** | Combined |

### What Sessions Revealed

Across 3 observed agent sessions (~7.5 hours, ~37 tickets):

- On every new/reassigned ticket, agents perform **4–6 manual copy-paste
  operations**: copy requester email from ticket body → paste into requester
  field → copy Org ID → paste into Org ID field → sometimes add followers
  by copying emails one at a time
- One agent manually typed the Org ID after looking it up via the
  Activations sidebar app — a 30+ second detour per ticket
- The email and Org ID are almost always present in the ticket body or
  previous messages — the information exists, it's just not parsed
- Some agents add 2+ follower emails by individual copy-paste from the
  ticket body, multiplying the friction

### The Human Cost

- **55 hours/month** of purely mechanical copy-paste work
- Error-prone: mistyped Org IDs lead to wrong account associations
- Delayed ticket routing when fields are missing
- Inconsistent data quality across agents (some skip fields under time
  pressure)
- New agents don't know which fields to populate or where to find the data

---

## 2. Architecture Overview

### System Diagram

```
┌────────────────────────────────────────────────────────────┐
│                         ZENDESK                            │
│                                                            │
│  ┌──────────────┐                                          │
│  │  New Ticket   │                                         │
│  │  Created /    │                                         │
│  │  Assigned     │                                         │
│  └──────┬───────┘                                          │
│          │                                                  │
│          ▼                                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              ZENDESK TRIGGER                          │  │
│  │  Conditions:                                          │  │
│  │  - Ticket is Created or Updated                       │  │
│  │  - Org ID field is blank                              │  │
│  │  - Tag "auto_populated" not present                   │  │
│  └──────────────┬───────────────────────────────────────┘  │
│                  │                                          │
│                  ▼                                          │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              MIDDLEWARE / SERVERLESS FUNCTION          │  │
│  │         (Cloud Function / Lambda / Zapier / n8n)      │  │
│  │                                                        │  │
│  │  1. Receive webhook (ticket_id)                        │  │
│  │  2. GET /api/v2/tickets/{id}.json                      │  │
│  │     → Read ticket body + description                   │  │
│  │  3. PARSE: extract email, Org ID, plan type            │  │
│  │     → Regex for Org ID: /o-[A-Z0-9]+-[a-z0-9]+/       │  │
│  │     → Regex for email: standard email pattern          │  │
│  │     → Keyword match for plan: "enterprise", "pro"      │  │
│  │  4. LOOKUP: validate Org ID via internal API or CRM    │  │
│  │     → GET org details (plan type, account status)      │  │
│  │  5. WRITE BACK: PUT /api/v2/tickets/{id}.json          │  │
│  │     → Set requester (if different from submitter)      │  │
│  │     → Set Org ID custom field                          │  │
│  │     → Set plan type (enterprise/pro/free)              │  │
│  │     → Set account status (active/trial/churned)        │  │
│  │     → Add tag "auto_populated"                         │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                 AGENT WORKFLOW (AFTER)                 │   │
│  │                                                        │   │
│  │  1. Open ticket → fields already populated             │   │
│  │  2. Verify at a glance (Org ID, plan, status visible)  │   │
│  │  3. Begin working on the actual issue                  │   │
│  │                                                        │   │
│  │  Old: 6 steps, ~4 min of copy-paste per ticket         │   │
│  │  New: 0 steps, fields pre-filled before agent opens    │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow

1. **Trigger**: New ticket created or existing ticket updated with empty
   Org ID field
2. **Webhook fires**: Zendesk sends ticket ID to middleware
3. **Parse**: Middleware reads ticket body/description and extracts email,
   Org ID, and keywords indicating plan type
4. **Validate**: Middleware looks up the Org ID against CRM/internal system
   to confirm it's valid and pull account metadata
5. **Write back**: Middleware updates the ticket via Zendesk API — sets
   requester, Org ID, plan type, account status, and adds `auto_populated`
   tag
6. **Agent opens ticket**: All fields are pre-filled, agent verifies at a
   glance and starts working

### Build vs Buy Decision

| Approach | Pros | Cons |
|----------|------|------|
| **Zendesk native triggers + Liquid markup** | No middleware needed, runs inside Zendesk | Limited parsing capability, can't do external lookups |
| **Custom middleware** (recommended) | Full parsing control, external CRM validation, handles edge cases | Requires hosting, more setup |
| **Zendesk App (sidebar)** | Can leverage Zendesk Apps Framework, runs in browser | Still requires agent action (clicking "populate"), not truly automated |
| **No-code** (Zapier/Make/n8n) | Fast to prototype, visual workflow builder | Per-execution costs at scale, less control over parsing logic |

**Recommendation**: Custom middleware for production (handles edge cases and
CRM validation). Prototype with Zapier/n8n first to validate the parsing
logic, then migrate to a serverless function.

---

## 3. Detailed Build Steps

### Prerequisites

- [ ] Zendesk Admin access (API tokens, triggers, custom fields)
- [ ] Cloud hosting account (GCP Cloud Functions, AWS Lambda, or similar)
- [ ] Access to CRM or internal system for Org ID validation (API key)
- [ ] Custom field IDs for Org ID, Plan Type, Account Status in Zendesk
      (Admin > Manage > Ticket Fields)

### Step 1: Document Custom Field IDs

1. Go to **Admin Center > Objects and Rules > Tickets > Fields**
2. Find and record the field IDs for:
   - `Org ID` (custom field) → e.g., `custom_field_12345`
   - `Plan Type` (custom field) → e.g., `custom_field_12346`
   - `Account Status` (custom field) → e.g., `custom_field_12347`
3. Note the expected values for dropdown fields (e.g., "enterprise",
   "pro", "free" for Plan Type)

### Step 2: Create Zendesk API Token

1. Go to **Admin Center > Apps and Integrations > Zendesk API**
2. Enable Token Access
3. Click **Add API Token**
4. Name it: `ticket-auto-populator`
5. Copy and store securely

### Step 3: Build the Parsing Function

**Core parsing logic (pseudocode):**

```javascript
function parseTicketBody(description, comments) {
  const fullText = [description, ...comments.map(c => c.body)].join('\n');

  // Extract Org ID — adapt regex to your org's ID format
  const orgIdMatch = fullText.match(/o-[A-Z0-9]+-[a-z0-9]+/i);
  const orgId = orgIdMatch ? orgIdMatch[0] : null;

  // Extract email — find requester email in body
  const emailMatch = fullText.match(
    /[\w.+-]+@[\w-]+\.[\w.-]+/
  );
  const email = emailMatch ? emailMatch[0] : null;

  // Extract plan type — keyword matching
  const planKeywords = {
    enterprise: /\b(enterprise|ent)\b/i,
    pro: /\b(pro|professional)\b/i,
    free: /\b(free|starter)\b/i,
  };
  let planType = null;
  for (const [plan, regex] of Object.entries(planKeywords)) {
    if (regex.test(fullText)) { planType = plan; break; }
  }

  return { orgId, email, planType };
}
```

### Step 4: Build the Validation & Lookup Function

```javascript
async function validateAndEnrich(orgId) {
  if (!orgId) return { valid: false };

  // Look up org in CRM / internal system
  const orgData = await crmClient.getOrganization(orgId);

  if (!orgData) return { valid: false, orgId };

  return {
    valid: true,
    orgId,
    planType: orgData.plan,          // "enterprise", "pro", etc.
    accountStatus: orgData.status,    // "active", "trial", "churned"
    orgName: orgData.name,
    csm: orgData.csm_email,
  };
}
```

### Step 5: Build the Write-Back Function

```javascript
async function populateTicketFields(ticketId, parsedData, enrichedData) {
  const updatePayload = {
    ticket: {
      custom_fields: [],
      tags: ['auto_populated'],
    },
  };

  // Set Org ID
  if (enrichedData.valid) {
    updatePayload.ticket.custom_fields.push({
      id: FIELD_IDS.ORG_ID,
      value: enrichedData.orgId,
    });
  }

  // Set Plan Type
  if (enrichedData.planType) {
    updatePayload.ticket.custom_fields.push({
      id: FIELD_IDS.PLAN_TYPE,
      value: enrichedData.planType,
    });
  }

  // Set Account Status
  if (enrichedData.accountStatus) {
    updatePayload.ticket.custom_fields.push({
      id: FIELD_IDS.ACCOUNT_STATUS,
      value: enrichedData.accountStatus,
    });
  }

  // Set requester if parsed email differs from current
  if (parsedData.email) {
    const currentTicket = await zendesk.getTicket(ticketId);
    const currentRequester = await zendesk.getUser(
      currentTicket.requester_id
    );
    if (currentRequester.email !== parsedData.email) {
      // Create or find user, then update requester
      const user = await zendesk.findOrCreateUser(parsedData.email);
      updatePayload.ticket.requester_id = user.id;
    }
  }

  await zendesk.updateTicket(ticketId, updatePayload);
}
```

**Zendesk API call:**
```
PUT /api/v2/tickets/{id}.json

{
  "ticket": {
    "custom_fields": [
      { "id": 12345, "value": "o-1RS5WZ-na1" },
      { "id": 12346, "value": "enterprise" },
      { "id": 12347, "value": "active" }
    ],
    "tags": ["auto_populated"]
  }
}
```

### Step 6: Create the Zendesk Webhook

1. Go to **Admin Center > Apps and Integrations > Webhooks**
2. Click **Create Webhook**
3. Configure:
   - **Name**: `Ticket Auto-Populator`
   - **Endpoint URL**: `https://{your-cloud-function-url}/populate`
   - **Request method**: POST
   - **Request format**: JSON
   - **Authentication**: Bearer token
4. Save

### Step 7: Create the Zendesk Trigger

1. Go to **Admin Center > Objects and Rules > Business Rules > Triggers**
2. Click **Create Trigger**
3. Configure:
   - **Name**: `Auto-Populate Ticket Fields on Creation`
   - **Conditions (ALL)**:
     - Ticket > Is > Created
     - Ticket > Tags > Does not contain > `auto_populated`
   - **Conditions (ANY)**:
     - Ticket > Org ID field > Is > (empty)
   - **Actions**:
     - Notify webhook > Ticket Auto-Populator
     - JSON body:
       ```json
       {
         "ticket_id": "{{ticket.id}}",
         "event": "ticket_created"
       }
       ```
4. Save and enable

### Step 8: Handle Edge Cases

| Scenario | Behavior |
|----------|----------|
| No Org ID found in body | Leave field blank, add tag `needs_org_id` for agent attention |
| Org ID found but invalid (not in CRM) | Leave field blank, add internal note: "Org ID '{id}' not found in CRM" |
| Multiple emails found | Use the first non-agent email |
| Ticket already has Org ID | Skip (trigger condition prevents firing) |
| Parsing error | Log error, leave fields unchanged, add tag `auto_populate_failed` |

---

## 4. Monitoring & Quality Tracking

**In FullStory (post-deployment):**
- "Input Org ID" event count (should decrease from 830/month baseline)
- Manual field population events on tickets tagged `auto_populated`
  (should be near zero — if agents are re-editing, parsing quality needs
  improvement)

**In middleware logs:**
- Successful population count per day
- Parse success rate (% of tickets where Org ID was found)
- CRM validation success rate (% of found Org IDs that are valid)
- Error rate and types
- Latency (should be < 2 seconds)

**In Zendesk:**
- Tickets with tag `auto_populated` vs `needs_org_id` vs
  `auto_populate_failed` (ratio shows coverage)
- Agent corrections: how often agents change auto-populated values
  (lower = better parsing accuracy)

---

## 5. Validation & Rollout

### Testing Checklist

- [ ] New ticket with Org ID in body → field auto-populated correctly
- [ ] New ticket with email in body → requester set correctly
- [ ] New ticket with no Org ID → tagged `needs_org_id`, field left blank
- [ ] New ticket with invalid Org ID → internal note posted, field blank
- [ ] Ticket already has Org ID → trigger does not fire (no double-write)
- [ ] Multiple emails in body → correct one selected
- [ ] Plan type and account status populated from CRM data
- [ ] Tag `auto_populated` added after successful population
- [ ] Error handling: API timeout, CRM unavailable, malformed body
- [ ] Performance: < 2 second latency from ticket creation to field update

### Phased Rollout

**Week 1: Shadow Mode**
- Deploy but log only — do NOT write back to Zendesk
- Compare parsed values against what agents manually enter
- Measure parse accuracy (target: > 90% match rate)

**Week 2-3: Write-Back Pilot**
- Enable write-back for one ticket group/category
- Agents verify auto-populated values and report mismatches
- Tune parsing regex and CRM lookup based on feedback

**Week 4+: Full Deployment**
- Enable for all new tickets
- Monitor `needs_org_id` tag volume (represents unparseable tickets)
- Iterate on parsing logic to reduce unparseable rate

### Rollback Plan

1. **Immediate**: Disable the Zendesk trigger (Admin > Triggers > disable)
2. **Full rollback**: Delete webhook and trigger; agents return to manual
   population with zero data loss

### Success Criteria

| Metric | Baseline | Target (90 days) |
|--------|----------|-------------------|
| "Input Org ID" events/month | 830 | < 100 (88% reduction) |
| Time per ticket intake | ~4 min | ~0 sec (automated) |
| Parse accuracy (Org ID) | N/A | > 95% |
| Agent corrections to auto-filled fields | N/A | < 5% |
| Tickets with missing Org ID | Current baseline | 50% reduction |

---

## 6. Cost Estimate

| Component | Estimated Monthly Cost |
|-----------|----------------------|
| Cloud Function hosting (~830 invocations/month) | ~$1–5 |
| CRM API calls (included in plan or negligible) | ~$0 |
| Zendesk API calls (included in plan) | $0 |
| **Total** | **~$1–5/month** |

Compared to **55 hours/month** of agent time saved, this is near-zero cost
for significant impact. This is one of the highest-ROI automations possible.

---

## 7. Adaptations for Other Orgs

| Platform | Trigger | Parsing Source | Field Write-Back |
|----------|---------|----------------|-----------------|
| Zendesk | Zendesk Trigger + Webhook | Ticket description/comments | Custom fields via API |
| Freshdesk | Automation Rule + Webhook | Ticket description | Custom fields via API |
| Intercom | Workflow + Webhook | Conversation first message | Contact/Company attributes via API |
| ServiceNow | Business Rule | Incident short_description + description | Fields via Table API |
| Salesforce Service Cloud | Flow / Process Builder | Case description + Web-to-Case fields | Case fields via API |
