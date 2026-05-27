# Implementation Blueprint: AI-Drafted Response Assembly

**Opportunity**: #1 — Response Assembly via Copy-Paste
**Platform**: Zendesk
**Automation Level**: AI-Assisted (AI drafts, agent reviews and sends)
**Estimated Savings**: 255 hours/month (5,093 events x 3 min each)
**Pattern**: AI Drafting

---

## 1. Current State

### What the Data Shows

| Signal | Monthly Volume | Source |
|--------|---------------|--------|
| Copied Previous Message | 1,612 | FullStory defined event |
| Copied Comment Log | 2,487 | FullStory defined event |
| Copied Text Editor | 994 | FullStory defined event |
| **Total copy-paste events** | **5,093** | Combined |

### What Sessions Revealed

Across 3 observed agent sessions (~7.5 hours, ~37 tickets):

- Agents spend **~42% of active time** composing replies
- The dominant pattern: open ticket → read history → copy fragments from
  previous messages and comment log → paste into reply composer → edit and
  stitch together → send
- One agent copied from history **5+ times** to assemble a single response
- One agent spent **9 minutes** reading + copy-pasting on a single ticket
- Knowledge Base accounts for only **0.4%** of all page views (126 out of
  34,455) — agents resolve tickets by copying from past ticket history
  instead of consulting KB articles

### The Human Cost

- **255 hours/month** of agent time spent assembling responses manually
- Inconsistent response quality (depends on which old ticket the agent finds)
- Slower resolution times (3+ min per ticket just for response assembly)
- New agents have no history to copy from — steep learning curve
- Tribal knowledge trapped in old ticket threads instead of searchable KB

---

## 2. Architecture Overview

### System Diagram

```
┌─────────────────────────────────────────────────────────┐
│                        ZENDESK                          │
│                                                         │
│  ┌──────────┐    Webhook     ┌────────────────────────┐ │
│  │  Ticket   │──────────────→│   Zendesk Webhook      │ │
│  │  Updated  │               │   (trigger on new      │ │
│  │           │               │    ticket or new        │ │
│  │           │               │    customer comment)    │ │
│  └──────────┘               └───────────┬────────────┘ │
│                                          │              │
│                                          ▼              │
│  ┌──────────────────────────────────────────────────┐   │
│  │              MIDDLEWARE LAYER                     │   │
│  │         (Cloud Function / Lambda / n8n)           │   │
│  │                                                   │   │
│  │  1. Receive webhook payload (ticket_id)           │   │
│  │  2. Fetch full ticket context via Zendesk API:    │   │
│  │     - Ticket body + all comments                  │   │
│  │     - Requester info + org metadata               │   │
│  │     - Ticket tags, category, priority             │   │
│  │  3. Search KB for relevant articles               │   │
│  │  4. Search recent similar resolved tickets        │   │
│  │  5. Build prompt with all context                 │   │
│  │  6. Call LLM API                                  │   │
│  │  7. Write draft back to Zendesk                   │   │
│  └──────────────┬───────────────────┬───────────────┘   │
│                  │                   │                    │
│                  ▼                   ▼                    │
│  ┌──────────────────┐   ┌──────────────────────────┐    │
│  │     LLM API      │   │   Zendesk API            │    │
│  │  (OpenAI / Claude │   │   - POST internal note   │    │
│  │   / Gemini)       │   │     with draft response  │    │
│  │                   │   │   - Agent sees draft,     │    │
│  │  Generates draft  │   │     edits, and sends      │    │
│  └──────────────────┘   └──────────────────────────┘    │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │                 AGENT WORKFLOW                     │    │
│  │                                                    │    │
│  │  1. Open ticket → AI draft already waiting as     │    │
│  │     internal note (marked with 🤖 prefix)         │    │
│  │  2. Review draft, edit if needed                   │    │
│  │  3. Copy to public reply (or one-click macro)      │    │
│  │  4. Send                                           │    │
│  │                                                    │    │
│  │  Old: 10 steps, ~3 min composing                   │    │
│  │  New: 3 steps, ~30 sec reviewing                   │    │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

### Data Flow

1. **Trigger**: Customer creates or updates a ticket (new comment)
2. **Webhook fires**: Zendesk sends ticket ID + event type to middleware
3. **Context assembly**: Middleware fetches full ticket thread, requester
   info, org metadata, KB articles, and similar resolved tickets
4. **AI generation**: LLM generates a draft response using assembled context
5. **Draft delivery**: Draft posted as an internal note on the ticket
6. **Agent review**: Agent opens ticket, sees draft, edits, and sends

### Build vs Buy Decision

| Approach | Pros | Cons |
|----------|------|------|
| **Zendesk AI Agent** (native) | Zero middleware, built-in, Zendesk-managed | Less customizable prompts, Zendesk pricing, limited context control |
| **Custom middleware** (recommended) | Full prompt control, any LLM, custom KB integration, audit trail | Requires hosting, more setup, you maintain it |
| **Third-party** (Forethought, Ada, etc.) | Turnkey, pre-built Zendesk integration | Vendor lock-in, monthly cost, less control over quality |

**Recommendation**: Start with custom middleware for maximum control over
response quality. Evaluate Zendesk AI Agent if they release better prompt
customization.

---

## 3. Detailed Build Steps

### Prerequisites

- [ ] Zendesk Admin access (to create webhooks, API tokens, and macros)
- [ ] Cloud hosting account (GCP Cloud Functions, AWS Lambda, or similar)
- [ ] LLM API key (OpenAI, Anthropic, or Google AI)
- [ ] Zendesk API token (Admin > Channels > API)

### Step 1: Create Zendesk API Token

1. Go to **Admin Center > Apps and Integrations > Zendesk API**
2. Enable Token Access
3. Click **Add API Token**
4. Name it: `ai-response-drafter`
5. Copy and store the token securely (you'll need it for the middleware)

Authentication format for API calls:
```
{email}/token:{api_token}
```

### Step 2: Build the Middleware Function

Create a serverless function (example: Google Cloud Function, Node.js):

**Function structure:**
```
ai-response-drafter/
├── index.js          # Main handler
├── zendesk.js        # Zendesk API client
├── llm.js            # LLM API client
├── prompt.js         # Prompt template builder
├── package.json      # Dependencies
└── .env              # API keys (use Secret Manager in production)
```

**Core logic (pseudocode):**

```javascript
async function handleWebhook(request) {
  const { ticket_id } = request.body;

  // 1. Fetch ticket context
  const ticket = await zendesk.getTicket(ticket_id);
  const comments = await zendesk.getTicketComments(ticket_id);
  const requester = await zendesk.getUser(ticket.requester_id);
  const org = ticket.organization_id
    ? await zendesk.getOrganization(ticket.organization_id)
    : null;

  // 2. Search KB for relevant articles
  const kbArticles = await zendesk.searchKB(ticket.subject + ' ' + ticket.description);

  // 3. Find similar resolved tickets
  const similarTickets = await zendesk.searchTickets(
    ticket.subject,
    { status: 'solved', limit: 3 }
  );

  // 4. Build prompt
  const prompt = buildPrompt({
    ticket, comments, requester, org, kbArticles, similarTickets
  });

  // 5. Call LLM
  const draft = await llm.generateResponse(prompt);

  // 6. Post draft as internal note
  await zendesk.addInternalNote(ticket_id, formatDraft(draft));

  return { status: 'ok', ticket_id };
}
```

### Step 3: Zendesk API Calls

**Get ticket with comments:**
```
GET /api/v2/tickets/{id}.json
GET /api/v2/tickets/{id}/comments.json
```

**Search Knowledge Base:**
```
GET /api/v2/help_center/articles/search.json?query={keywords}
```

**Search similar resolved tickets:**
```
GET /api/v2/search.json?query=type:ticket status:solved {keywords}
```

**Post internal note (the AI draft):**
```
PUT /api/v2/tickets/{id}.json

{
  "ticket": {
    "comment": {
      "body": "🤖 AI Draft Response:\n\n{draft_text}\n\n---\nSources: {kb_article_links}\nSimilar tickets: {similar_ticket_ids}",
      "public": false
    }
  }
}
```

### Step 4: LLM Prompt Template

```
You are a customer support agent for {company_name}. Draft a response
to the customer's latest message on this support ticket.

## Ticket Context
- Subject: {ticket.subject}
- Priority: {ticket.priority}
- Category: {ticket.category}
- Customer: {requester.name} ({requester.email})
- Organization: {org.name} (Plan: {org.plan_type})

## Conversation History (most recent first)
{formatted_comments}

## Relevant Knowledge Base Articles
{formatted_kb_articles}

## Similar Resolved Tickets
{formatted_similar_tickets}

## Instructions
- Draft a professional, empathetic response to the customer's latest message
- Reference relevant KB articles when applicable (include links)
- If this is a known issue with a documented resolution, provide the steps
- Match the tone and style of previous agent responses in this thread
- If you need more information from the customer, ask specific questions
- Keep the response concise but complete
- Do NOT make up information — if you're unsure, say so and suggest
  the agent verify before sending
- Format with clear paragraphs, bullet points for steps, and links
  where helpful

## Draft Response:
```

### Step 5: Create the Zendesk Webhook

1. Go to **Admin Center > Apps and Integrations > Webhooks**
2. Click **Create Webhook**
3. Configure:
   - **Name**: `AI Response Drafter`
   - **Endpoint URL**: `https://{your-cloud-function-url}/draft`
   - **Request method**: POST
   - **Request format**: JSON
   - **Authentication**: Bearer token (create a shared secret)
4. Save

### Step 6: Create the Zendesk Trigger

1. Go to **Admin Center > Objects and Rules > Business Rules > Triggers**
2. Click **Create Trigger**
3. Configure:
   - **Name**: `AI Draft Response on New Customer Comment`
   - **Conditions (ALL)**:
     - Ticket > Status > Is not > Solved
     - Ticket > Comment > Is > Present, and requester is > (end user)
     - Ticket > Tags > Does not contain > `ai_draft_skip`
   - **Actions**:
     - Notify webhook > AI Response Drafter
     - JSON body:
       ```json
       {
         "ticket_id": "{{ticket.id}}",
         "event": "customer_comment"
       }
       ```
4. Save and enable

### Step 7: Create Agent Macros

Create macros to streamline the agent's workflow with AI drafts:

**Macro 1: "Use AI Draft"**
- Action: Copies the most recent internal note (AI draft) to the
  public reply field
- This can be implemented as a Zendesk app or a macro that
  references the latest internal note

**Macro 2: "Skip AI Draft"**
- Action: Adds tag `ai_draft_skip` to the ticket
- Use when the agent wants to compose manually (e.g., sensitive topics)

**Macro 3: "Submit as Pending + 5-Day Follow-up"**
- Combines: Set status to Pending + Set follow-up timer to 5 days +
  Set follow-up type to Manual
- Replaces the 5-click field population ritual (bundles with
  automation candidate #3)

### Step 8: Monitoring & Quality Tracking

Track these metrics to measure success:

**In FullStory (post-deployment):**
- Copy-paste event count (should decrease from 5,093/month baseline)
- Time spent composing replies (should decrease from 42% of active time)
- Agent active time per ticket (should decrease)

**In middleware logs:**
- Draft generation count per day
- LLM latency (p50, p95)
- Error rate (failed API calls, timeouts)
- Token usage and cost

**In Zendesk (manual or via API):**
- Draft acceptance rate: % of tickets where agents use the AI draft
  (vs. composing from scratch)
- Draft edit distance: how much agents modify the draft before sending
  (lower = better quality)
- First response time: should improve
- Customer satisfaction (CSAT): should remain stable or improve

---

## 4. Validation & Rollout

### Testing Checklist

- [ ] Webhook fires on new customer comment (verify in webhook logs)
- [ ] Middleware receives payload and fetches correct ticket data
- [ ] KB search returns relevant articles (test with 5 known topics)
- [ ] Similar ticket search returns useful results
- [ ] LLM generates appropriate, on-brand responses
- [ ] Draft appears as internal note on the correct ticket
- [ ] Draft does NOT appear for solved tickets or agent-only comments
- [ ] Error handling: test with invalid ticket ID, API timeout, LLM error
- [ ] Rate limiting: verify behavior under burst load (10+ tickets/min)
- [ ] AI draft is clearly marked as AI-generated (agents can distinguish)

### Phased Rollout

**Week 1-2: Shadow Mode (2 pilot agents)**
- Deploy to production but only for 2 volunteer agents
- AI drafts appear as internal notes — agents are NOT required to use them
- Agents provide qualitative feedback on draft quality
- Monitor: draft generation success rate, agent feedback

**Week 3-4: Expanded Pilot (full team, optional use)**
- Enable for all agents
- Still optional — agents can use or ignore drafts
- Track: draft acceptance rate, edit distance, CSAT
- Refine prompt based on patterns in agent edits

**Week 5+: Default Workflow**
- AI drafts become the standard starting point
- Agents still have full control to edit or compose from scratch
- Track ongoing metrics vs baseline

### Rollback Plan

If issues arise:
1. **Immediate**: Disable the Zendesk trigger (Admin > Triggers > disable)
2. **Graceful**: Add tag `ai_draft_skip` to all new tickets via a
   catch-all trigger (drafts stop but no data is lost)
3. **Full rollback**: Delete the webhook and trigger; agents return to
   manual workflow with zero impact

### Success Criteria

| Metric | Baseline | Target (90 days) |
|--------|----------|-------------------|
| Copy-paste events/month | 5,093 | < 1,500 (70% reduction) |
| Time composing replies | 42% of active time | < 20% |
| Agent response time | Current average | 30% faster |
| Draft acceptance rate | N/A | > 60% |
| CSAT score | Current baseline | No decrease |

---

## 5. Cost Estimate

| Component | Estimated Monthly Cost |
|-----------|----------------------|
| LLM API (GPT-4o at ~1K tokens/call x 5,093 calls) | ~$25–50 |
| Cloud Function hosting | ~$5–10 |
| Zendesk API calls (included in plan) | $0 |
| **Total** | **~$30–60/month** |

Compared to **255 hours/month** of agent time saved, this is a
significant ROI even at modest agent hourly rates.

---

## 6. Adaptations for Other Orgs

This blueprint assumes Zendesk, but the same pattern applies to any
support platform:

| Platform | Webhook Trigger | API for Context | Draft Delivery |
|----------|----------------|-----------------|----------------|
| Zendesk | Zendesk Trigger + Webhook | Zendesk REST API v2 | Internal note via API |
| Intercom | Intercom Webhook (conversation.user.replied) | Intercom API | Admin note via API |
| Freshdesk | Freshdesk Automation Rule + Webhook | Freshdesk API v2 | Private note via API |
| ServiceNow | Business Rule + REST Message | ServiceNow Table API | Work note via API |
| Salesforce Service Cloud | Process Builder / Flow + Platform Event | Salesforce REST API | Chatter post or Case Comment |

The middleware logic, LLM prompt template, and rollout strategy
remain the same regardless of platform.
