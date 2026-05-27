# Running Workforce Automation Analysis with Gemini

This guide covers how to run the FullStory Workforce Automation Analysis
using Gemini (either Gemini CLI locally or via Vertex AI on Google Cloud)
instead of Cursor.

---

## Option 1: Gemini CLI (Recommended)

Gemini CLI runs locally and has native MCP support via extensions.
The FullStory plugin already ships with a `gemini-extension.json`.

### Prerequisites

- [Gemini CLI](https://github.com/google-gemini/gemini-cli) installed
- A FullStory account enrolled in the [MCP beta program](https://www.fullstory.com/platform/mcp/)
- Node.js 18+ (for Gemini CLI)

### Step 1: Install the FullStory Extension

The FullStory MCP extension can be installed from the plugin registry.
Once installed, the MCP server config is:

```json
{
  "name": "fullstory",
  "version": "0.1.0",
  "mcpServers": {
    "fullstory": {
      "type": "http",
      "url": "https://api.fullstory.com/mcp/fullstory",
      "oauth": {
        "scopes": [
          "search:read",
          "search.metadata:read",
          "sessions:read",
          "metrics:read",
          "metrics.funnel:read",
          "library.objects:read",
          "playback.spec:read",
          "playback.page.definitions:read"
        ]
      }
    }
  }
}
```

### Step 2: Switch to Staging

To point at the **staging** environment instead of production, modify the
MCP server URL in the extension config. Replace:

```
"url": "https://api.fullstory.com/mcp/fullstory"
```

With the staging equivalent (confirm the exact URL with your team):

```
"url": "https://api.staging.fullstory.com/mcp/fullstory"
```

Or if staging uses a different pattern:

```
"url": "https://staging-api.fullstory.com/mcp/fullstory"
```

### Step 3: Authenticate

On first use, Gemini CLI will open a browser window for OAuth
authentication with FullStory. Log in with your FullStory staging
credentials.

### Step 4: Run the Analysis

Paste the contents of `gemini-analysis-playbook.md` (included in this
skill directory) as your opening prompt. The playbook is adapted for
Gemini's tool naming conventions.

### Tool Name Mapping

In Gemini CLI, MCP tools follow the pattern `mcp_<server>_<toolName>`.
With the server key `fullstory`, the mapping is:

| Cursor Tool Name            | Gemini CLI Tool Name                    |
|-----------------------------|-----------------------------------------|
| `fullstory:build_metric`    | `mcp_fullstory_build_metric`            |
| `fullstory:compute_metric`  | `mcp_fullstory_compute_metric`          |
| `fullstory:build_segment`   | `mcp_fullstory_build_segment`           |
| `fullstory:update_metric`   | `mcp_fullstory_update_metric`           |
| `fullstory:update_segment`  | `mcp_fullstory_update_segment`          |
| `fullstory:get_metric`      | `mcp_fullstory_get_metric`              |
| `fullstory:get_segment`     | `mcp_fullstory_get_segment`             |
| `fullstory:get_sessions`    | `mcp_fullstory_get_sessions`            |
| `fullstory:get_session_events` | `mcp_fullstory_get_session_events`   |
| `fullstory:get_pages`       | `mcp_fullstory_get_pages`               |
| `fullstory:discover_org_context` | `mcp_fullstory_discover_org_context` |
| `fullstory:get_view_counts` | `mcp_fullstory_get_view_counts`         |

---

## Option 2: Vertex AI (vertexsearch.cloud.google.com)

Vertex AI's web interface can use Gemini models with extensions. There
are two sub-approaches depending on what's available in your environment.

### 2A: Vertex AI Agent Builder with MCP

If your Vertex AI environment supports MCP server connections:

1. **Create a new Agent** in Vertex AI Agent Builder
2. **Add the FullStory MCP as a tool/extension**:
   - Type: HTTP / Streamable HTTP
   - URL: `https://api.fullstory.com/mcp/fullstory` (or staging URL)
   - Authentication: OAuth 2.0 with the scopes listed above
3. **Add the analysis playbook as system instructions**:
   - Copy the contents of `gemini-analysis-playbook.md` into the
     agent's system prompt / instructions field
4. **Start a conversation** and say:
   "Run a workforce automation analysis for this org"

### 2B: Manual Prompt Approach (No MCP)

If Vertex AI Search doesn't support direct MCP connections, you can
still use the methodology manually:

1. **Open Vertex AI Search** at vertexsearch.cloud.google.com
2. **Use the FullStory API directly** alongside the Gemini conversation:
   - Run the MCP queries via the FullStory API or a local Gemini CLI
     session in parallel
   - Paste the results into the Vertex AI conversation for analysis
3. **Use the playbook** as a structured guide for what to query and
   in what order

This is less automated but still follows the same analytical framework.

### 2C: Vertex AI with Gemini CLI as Backend

The most practical approach may be a hybrid:

1. Use **Gemini CLI locally** for data gathering (Phases 0–2) since it
   has native MCP support and can call FullStory tools directly
2. Export the findings as structured data
3. Use **Vertex AI** for synthesis and report generation (Phases 3–4)
   where you need the larger context window and richer output

---

## Key Differences from Cursor

| Capability | Cursor | Gemini CLI | Vertex AI Web |
|-----------|--------|-----------|---------------|
| MCP tool calls | Native | Native (via extensions) | Via Agent Builder or manual |
| Subagents for session analysis | Task tool (parallel) | Sequential (no subagents) | Sequential |
| Skill/playbook loading | Automatic (SKILL.md) | Manual (paste or GEMINI.md) | System instructions |
| File writing (HTML report) | Native | Native | Copy from output |
| Session replay viewing | Session open/view/diff tools | Same tools available | Same if MCP connected |

### Adapting for No Subagents

Cursor uses subagents to analyze multiple sessions in parallel. Without
subagents, call `get_session_events` sequentially for each session.
To manage context window limits:

1. Analyze one session at a time
2. Summarize findings before moving to the next session
3. Ask the model to retain only the structured findings summary,
   not the raw event transcript

---

## Staging vs Production

| Environment | MCP URL (confirm with your team) |
|------------|-------------------------------|
| Production | `https://api.fullstory.com/mcp/fullstory` |
| Staging | `https://api.staging.fullstory.com/mcp/fullstory` (verify) |
| Playpen | TBD — check internal docs |
| Local Dev | TBD — check internal docs |

The staging MCP URL may differ from the pattern above. Check with
the FullStory MCP team or your internal `/enable-fullstory-mcp` command
source code to find the exact staging endpoint.

---

## Quick Start Checklist

- [ ] Confirm the staging MCP URL with your team
- [ ] Install Gemini CLI (or configure Vertex AI Agent Builder)
- [ ] Configure the FullStory MCP extension with the staging URL
- [ ] Authenticate via OAuth (browser popup on first use)
- [ ] Paste the analysis playbook as your opening prompt
- [ ] Say: "Run a workforce automation analysis for this org"
