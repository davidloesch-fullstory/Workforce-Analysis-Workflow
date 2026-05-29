---
name: instrumentation-spec
description: >-
  Generate a FullStory pre-flight instrumentation spec for a tool an org wants
  to evaluate. Researches the tool's real workflows and surfaces, then recommends
  the pages, named elements, defined events, and custom properties a FullStory
  admin should configure BEFORE a workforce analysis — so the analysis runs on
  high-precision, action-level signal instead of URL-path inference. Produces a
  single self-contained, tabbed HTML file (Overview + one tab per tool + a team
  rollup tab). Use when a user names a tool (or team) they want to evaluate and
  asks what to configure / instrument in FullStory first.
---

# Instrumentation Spec Generator (Pre-Flight Config)

Produces a build sheet that tells a FullStory admin exactly what to configure
for a target tool **before** a workforce automation analysis runs. Configuring
these definitions promotes a tool's signal from low-precision URL-path inference
to high-precision, action-level events — which is what eliminates the
URL-tokenization guesswork that otherwise undercounts real workflows.

The output is a single self-contained, tabbed HTML file: an Overview tab, one
tab per tool (the deep spec), and a Team Rollup tab that consolidates a team's
tools by workflow. A business or platform leader can review everything in one
place without switching documents.

## The Boundary (read first)

FullStory's analysis tooling can **discover** and **query** definitions
(`discover_org_context`, `get_pages`, `build_metric`) but it **cannot create
them**. There is no create-element / create-event MCP tool. The definitions in
this spec are authored by a human admin in **Data Studio** (Data Management).

So this skill produces a *plan*; the human applies it; the later analysis
consumes the result:

```
Research tool → generate spec (this skill) → ADMIN configures in FullStory
  → capture window (~2-4 weeks) → workforce analysis queries the new signals
```

Always state this handoff explicitly in the deliverable. Never imply the agent
configures FullStory directly.

## When to Use This Skill

- User names a tool an org wants to evaluate and asks "what should we configure
  / instrument in FullStory?"
- User wants richer signal for an upcoming workforce analysis ("set this up so
  the analysis is more precise")
- User asks for a pre-flight / Phase-0 instrumentation plan for a team
- Run it tool-by-tool (deep spec for one tool) or team-wide (every tool a team
  uses, rolled up) — see Scope below

**Scope note**: Whether speccing one tool or a whole team, the output is always
ONE self-contained tabbed HTML file — Overview tab + one tab per tool + a Team
Rollup tab. A single-tool request still gets the rollup tab (with future tools
marked as not-yet-spec'd). Never emit separate files per tool.

## Depth: Conceptual, Not Selector-Level

FullStory named elements bind to **CSS selectors**, which are instance- and
surface-specific (and can be volatile in modern component-driven UIs). This
skill therefore works at **conceptual depth**: name the action and *where it
lives* (the surface + control), and leave the actual selector / URL-rule binding
to the admin, who confirms each one in-instance using Inspect Mode on a real
session. Do not invent CSS selectors.

## Inputs Required

Before starting, confirm:

1. **The tool(s)** — name, and ideally the product/edition in use (tools like
   ServiceNow span many products; scope to the relevant one, e.g. ITSM Incident).
2. **The team** — which team will be analyzed, for the rollup tab and for
   prioritizing the workflows that matter.
3. **Surfaces in use** — if the tool has multiple UIs (e.g. a modern workspace
   vs. a classic UI), confirm which ones agents actually work in, since elements
   must be named per surface.

If unclear, ask before proceeding.

## The FullStory Definition Taxonomy

Every recommendation maps to one of four FullStory definition types. Use these
consistently and tag them with the matching pill in the HTML:

| Type | What it is | FullStory home | Pill |
|------|-----------|----------------|------|
| **Page** | A URL-rule grouping that names a meaningful destination/view | Settings → Data Management → Pages (rules use `*`, `**`, `[]`) | `pill-accent` |
| **Named Element** | A name bound to a CSS selector for a specific control | Data Studio via Inspect Mode (`element_definition_id`) | `pill-info` |
| **Defined Event** | A higher-level "user did X" signal, built from element + page criteria | Settings → Data Management → Events | `pill-success` |
| **Custom Property** | A dimension attached to an event for slicing | Set on event/element context | `pill-neutral` |

## Spec Structure

Every per-tool tab follows 6 sections. The snippets below describe the *content*
each section must contain — they are NOT the output format. The output is HTML,
assembled from `instrumentation-spec-template.html`. Use
`instrumentation-specs/servicenow.html` as the copy-from reference.

### Section 1: Surfaces & The Discipline They Force
- Enumerate the tool's UIs (e.g. modern workspace, classic UI, mobile) and which
  objects/views live where.
- Call out that the same underlying object may render through multiple surfaces,
  so named elements are scoped per surface.
- Include the "why conceptual depth" callout (CSS-selector binding + volatility),
  cited to FullStory + the vendor's UI docs.

### Section 2: Pages to Define `pill-accent`
- Table: Page name | illustrative URL rule | why it matters | verify badge.
- URL patterns are illustrative; vendor URLs are instance-specific — mark them
  `verify-unverified` (instance-specific) and tell the admin to confirm.
- Cite FullStory's URL-rule syntax doc.

### Section 3: Named Elements to Capture `pill-info`
- Table: element | where it lives (surface + control) | action it represents.
- Cite the vendor doc that establishes the control/field exists.

### Section 4: Defined Events `pill-success` (the core)
- Table: Defined Event | built from (conceptual) | workflow measured |
  automation hypothesis it supports | maps to native vendor automation.
- The last column ties each event to the vendor's own automation capability
  (with plan-tier gating) so the later analysis can frame build-vs-buy.
- Follow with a native-automation caveat callout (plan-gating, tiers).

### Section 5: Custom Properties to Capture `pill-neutral`
- Table: property | on which events | why (what slice it enables).

### Section 6: How to Configure in FullStory
- The admin handoff, as numbered build steps (reuse the `.build-step` component):
  Pages → Elements → Events (order matters — events reference the others).
- Each step carries a `verify-badge` and a `<div class="src">` link to the
  FullStory help doc for that action.
- End with a `.callout-success`: let it capture ~2-4 weeks, then the analysis
  discovers these by name and builds metrics on them (enrichment → primary).

### Team Rollup Tab
- Workflow-coverage table: workflow | primary tool | FullStory signal(s) | spec
  status (Spec'd / Future tool spec).
- Dedup & double-count guardrails: one workflow = one canonical event source;
  cross-tool handoffs modeled as a single linkage event; per-tool active time is
  a directional floor, never summed across overlapping tabs.

## Grounding & Verification (mandatory)

Before writing tool specifics, look up current vendor documentation with the web
tools (`WebSearch` / `WebFetch`). Confirm:

- The tool's UI surfaces and the objects/fields/controls referenced
- Native automation capabilities and their **plan-tier gating** (and current
  packaging — vendors repackage; verify the live tier names)
- The FullStory configuration mechanics for each definition type

Rules:
- Cite each verified fact with `<div class="src">Source: <a href="{url}">{title}</a></div>`
  (or an inline `<span class="src">`).
- Tag items with a verification badge: `verify-verified` (confirmed against
  current docs), `verify-unverified` (instance-specific or unconfirmed — say so),
  `verify-plangated` (requires a specific plan/tier).
- Never present an assumed surface, field, URL rule, or plan tier as confirmed.
- Always cite the FullStory side too (how the admin defines pages/elements/events).

## Process

For a team-wide request, run steps 1-7 once per tool, then assemble all tool tabs
plus the rollup into a single tabbed file in step 8.

1. **Confirm inputs** — tool(s), edition/product, team, surfaces in use.
2. **Research the tool (grounded)** — identify the 5-10 workflows where the work
   actually happens, the surfaces they live on, and the controls/fields involved.
   Use `WebSearch` / `WebFetch`; cite as you go.
3. **Map to FullStory definitions** — for each workflow, derive the Pages,
   Named Elements, Defined Events, and Custom Properties that capture it. Keep
   it conceptual (action + surface), not selector-level.
4. **Verify native automation + plan-gating** — for each defined event, identify
   the vendor's own automation capability that could address it and its tier.
   This powers the build-vs-buy framing.
5. **Write Sections 1-5** for the tool tab (surfaces, pages, elements, events,
   properties), each grounded and badged.
6. **Write Section 6 (How to Configure)** — the admin handoff build steps with
   FullStory help-doc citations.
7. **Write the Team Rollup** — coverage table + dedup guardrails. Mark tools not
   yet spec'd as "Future tool spec."
8. **Assemble and save the tabbed HTML** — copy the shell from
   `instrumentation-spec-template.html`, fill the Overview tab (hero counts of
   pages/elements/events/properties, the configure→capture→analyze flow SVG, the
   human-handoff callout, one pitch card per tool + the rollup), then write to
   `deliverables/{prefix}-instrumentation-spec.html` where `{prefix}` matches the
   engagement (e.g. `WFA-FSWorkforce-Support-2026-05`). One file per engagement.
   Specs are generated on demand and are NOT persisted as separate Markdown files.

## Output Format

A **single self-contained, tabbed HTML file** — Overview tab + one tab per tool +
a Team Rollup tab. It shares the styling of the blueprint and Phase-4 analysis
deliverables so everything reads as one family.

**Construction**:
- Copy the shell, inline CSS, tab logic, and SVG flow system from
  `instrumentation-spec-template.html`. Everything inline — no external assets.
- Each tool tab contains the 6 sections in order.
- The Overview tab shows hero counts (pages / named elements / defined events /
  custom properties), the configure→capture→analyze SVG flow, the human-handoff
  callout, and one clickable pitch card per tool plus the rollup.
- Use `instrumentation-specs/servicenow.html` as the gold standard for tone,
  depth, structure, styling, and the SVG flow diagram.

**Calibration**:
- Conceptual depth — name the action and surface; don't invent selectors.
- Every tool specific and plan tier carries a source link + verification badge.
- Use HTML tables for the page/element/event/property lists, not prose.
- Use the `.build-step` component for the configuration handoff.

## Reference Implementation

See `instrumentation-specs/servicenow.html` for a complete example covering:
- ServiceNow ITSM (Incident) across Service Operations Workspace + classic UI16
- 6 pages, 12 named elements, 9 defined events, 10 custom properties
- Native-automation mapping to ServiceNow's agentic workflows with Foundation /
  Advanced / Prime plan-gating
- The shared tabbed shell, the configure→capture→analyze SVG flow, source-linked
  configuration steps with verification badges, and the IT Service Desk rollup.

This reference demonstrates the target quality, depth, format, and styling for
all instrumentation specs.
