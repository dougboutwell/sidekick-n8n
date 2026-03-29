# Sidekick (n8n)

Workflow automation via n8n, exposed to Claude Code as an MCP server. Successor to [dougboutwell/sidekick](https://github.com/dougboutwell/sidekick) — same problems, but using n8n as the execution layer instead of custom TypeScript.

## Running n8n

```bash
docker compose up -d
```

n8n UI at http://localhost:5678. Data persists in `~/.n8n/` via Docker volume mount.

To stop: `docker compose down`. Data survives across restarts.

## Claude Code Connection

Add n8n as an MCP server (already done for the sidekick project):

```bash
claude mcp add n8n http://localhost:5678/mcp-server/http \
  -t http -s project \
  -H "Authorization: Bearer <MCP_API_KEY>"
```

The MCP API key is generated in n8n UI: **Settings > MCP Access**. This is a separate key from the public API key (audience: `mcp-server-api`).

## Building Workflows

### Via MCP tools (preferred)

Claude Code can build workflows programmatically through n8n's MCP tools:

1. `search_nodes` — find node types for services you need
2. `get_node_types` — get exact parameter definitions (MUST do this, don't guess)
3. `validate_workflow` — validate SDK code before creating
4. `create_workflow_from_code` — save to n8n
5. `publish_workflow` — activate for production use
6. `execute_workflow` — run and get results
7. `get_execution` — fetch full execution results

### SDK syntax

n8n's Workflow SDK uses factory functions, not constructors:

```javascript
import { workflow, trigger, node, newCredential } from '@n8n/workflow-sdk';

const myTrigger = trigger({
  type: 'n8n-nodes-base.webhook',
  version: 2.1,
  config: { name: 'My Trigger', parameters: {...}, position: [240, 300] },
  output: [{ example: 'data' }],
});

const myNode = node({
  type: 'n8n-nodes-base.jira',
  version: 1,
  config: {
    name: 'Query Jira',
    parameters: { resource: 'issue', operation: 'getAll', ... },
    credentials: { jiraSoftwareCloudApi: newCredential('Jira') },
    position: [540, 300],
  },
  output: [{ key: 'PDDD-1234' }],
});

export default workflow('my-workflow', 'My Workflow')
  .add(myTrigger)
  .to(myNode);
```

Read the full SDK reference via MCP resource: `n8n://workflow-sdk/reference`

### Via UI

Open http://localhost:5678, build visually. Good for debugging and iteration.

## Workflow Files

Exported workflow JSON lives in `workflows/`. These are reference copies for version control — the live workflows run from n8n's SQLite database in `~/.n8n/`.

Note: workflow JSON does NOT include credentials. On import, n8n auto-assigns credentials if matching types exist in the credential store.

## Credentials

Managed in n8n UI: **Settings > Credentials**. Encrypted at rest (AES) in `~/.n8n/database.sqlite`.

Currently configured:
- Jira Software Cloud (API token)
- Anthropic (API key)

## n8n API

For operations not exposed via MCP, the REST API is at `http://localhost:5678/api/v1/`. Requires `X-N8N-API-KEY` header. Generate key in **Settings > API**.

Key endpoints:
- `POST /api/v1/workflows` — create workflow from JSON
- `PUT /api/v1/workflows/:id` — update workflow (also used to activate: `{"active": true}`)
- `POST /api/v1/workflows/:id/activate` — activate workflow

## Architecture

```
Claude Code ──MCP──> n8n MCP Server (/mcp-server/http)
                         │
                         ├── execute_workflow ──> Workflow Engine
                         │                           │
                         │                           ├── Jira nodes (native integration)
                         │                           ├── AI Agent nodes (LLM reasoning)
                         │                           ├── Code nodes (custom logic)
                         │                           └── Webhook/Schedule triggers
                         │
                         ├── search_workflows ──> Workflow metadata
                         ├── create_workflow_from_code ──> Workflow SDK
                         └── get_execution ──> Execution results
```

## Footguns

### n8n SDK

- Factory functions (`node()`, `trigger()`), NOT constructors (`new Node()`)
- Every node MUST have an `output` property with sample data
- Use `expr('{{ $json.field }}')` for expressions, NOT template literals
- Use `newCredential('Name')` for auth, NEVER hardcoded tokens
- Use `placeholder('hint')` directly as parameter value, not inside `expr()`

### MCP server

- Endpoint is `/mcp-server/http`, NOT `/mcp/<path>`
- MCP API key (audience: `mcp-server-api`) is separate from public API key
- Requires `Accept: application/json, text/event-stream` header

### Workflow execution

- MCP Server Trigger workflows can't be executed via `execute_workflow` — they're for receiving MCP tool calls
- Webhook/Schedule/Manual trigger workflows CAN be executed via `execute_workflow`
- `execute_workflow` returns `executionId` — call `get_execution` to get full results

### Credentials in workflow JSON

- Exported JSON does NOT contain credentials
- On import/create, n8n auto-assigns matching credential types if they exist
- `autoAssignedCredentials` in the create response shows what was matched

## Context: Why n8n

This repo replaces [dougboutwell/sidekick](https://github.com/dougboutwell/sidekick), which was a custom MCP facade in TypeScript. The custom engine (~200 LOC) handled tool routing, upstream MCP connections, agent execution via Vercel AI SDK, and module loading from YAML/markdown manifests.

The problem: every feature on the roadmap (OAuth, HTTP transport, credential storage, Glean integration, visual debugging, workflow distribution) was converging on rebuilding what n8n already ships. The auth layer alone would have been ~200 more lines, and that's before OAuth token refresh, credential encryption, or a UI for managing secrets.

n8n provides:
- MCP server AND client (both streamable HTTP and SSE)
- OAuth2 with automatic token refresh
- Credential store (AES-encrypted at rest)
- AI Agent nodes with tool filtering and structured output
- Visual workflow builder and debugger
- Native integrations for Jira, Slack, and hundreds of others
- Self-hosted, open source

What we lost: code-first YAML manifests, precise Zod schema enforcement via `Output.object()`, lightweight `npx` deployment. What we gained: everything else, maintained by someone else.
