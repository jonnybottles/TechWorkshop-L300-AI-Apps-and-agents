# Azure Container App Deployment Issues

## Environment
- **Container App**: `t25lbln64kquk-app`
- **Resource Group**: `rg-tw-l300-agents`
- **Managed Identity Principal ID**: `64d0b324-9ea1-4d87-bc1e-f786b438684c`
- **AI Foundry Account**: `aif-t25lbln64kquk`
- **AI Foundry Project**: `proj-t25lbln64kquk`
- **App URL**: `https://t25lbln64kquk-app.victoriouspond-065359d6.eastus2.azurecontainerapps.io`

## Current Role Assignments on Managed Identity
| Role | Scope |
|------|-------|
| AcrPull | Container Registry `t25lbln64kqukcosureg` |
| Cognitive Services OpenAI User | AI Foundry account `aif-t25lbln64kquk` |
| Cognitive Services OpenAI User | AI Foundry project `proj-t25lbln64kquk` |
| Azure AI User | Resource group `rg-tw-l300-agents` |
| Azure AI Developer | AI Foundry account `aif-t25lbln64kquk` |
| Azure AI Developer | AI Foundry project `proj-t25lbln64kquk` |

## Issue 1: Inventory Agent — "Connection closed" error

**Status**: Partially working — agent routes correctly but MCP tool call fails

**Symptom**: When asking "Can you check if PROD0045 is in stock?", the handoff service correctly routes to `inventory_agent`, but the agent's tool call to `mcp_inventory_check` returns a "Connection closed" error.

**Response received**:
```json
{
  "answer": "I couldn't retrieve the inventory for PROD0045 because the inventory service returned an error: \"Connection closed\".",
  "agent": "inventory_agent"
}
```

**Likely cause**: The MCP inventory server runs as a subprocess via stdio transport (`mcp_inventory_client.py`). In the Docker container, the MCP server process may not be starting correctly or the stdio connection is being dropped. This could be:
- The MCP server subprocess failing to launch in the container
- A missing dependency or path issue inside the container
- The async stdio connection timing out or closing prematurely

**Files to investigate**:
- `src/app/servers/mcp_inventory_client.py` — manages the persistent stdio connection to the MCP server
- `src/app/servers/mcp_inventory_server.py` — the FastMCP server that wraps the tool implementations
- `src/app/agents/mcp_tools.py` — async wrappers that call the MCP client

## Issue 2: Customer Loyalty Agent — PermissionDenied

**Status**: Not working — permission error persists despite role assignments

**Symptom**: The customer loyalty background task (runs on first cart operation) returns a PermissionDenied error. This happens even though other agents (interior_designer, cart_manager) work fine.

**Response received**:
```json
{
  "answer": "{'error': {'code': 'PermissionDenied', 'message': 'Principal does not have access to API/Operation.'}}",
  "agent": "customer_loyalty"
}
```

**Likely cause**: The customer loyalty agent calls the `mcp_calculate_discount` tool, which internally calls `calculate_discount()` in `src/app/tools/discountLogic.py`. That function creates its own Azure OpenAI client to call the GPT model for discount calculation. This secondary OpenAI call may be using a different authentication path or endpoint that the managed identity doesn't have access to.

**Files to investigate**:
- `src/app/tools/discountLogic.py` — creates its own OpenAI client; check how it authenticates (API key vs DefaultAzureCredential vs managed identity)
- Check if `discountLogic.py` uses `gpt_endpoint` and `gpt_api_version` env vars with `DefaultAzureCredential` — the managed identity may need `Cognitive Services OpenAI Contributor` instead of just `User`

## What's Working
- **Interior Designer Agent**: Routes correctly, returns product recommendations with images and prices
- **Cart Manager Agent**: Routes correctly, manages cart state, handles checkout with Miami pickup location
- **Handoff Service**: Intent classification and agent routing works correctly across all domains
- **Cora Agent**: General shopping queries work

## Summary of What Needs Fixing
1. Fix the MCP stdio connection issue inside the Docker container so the inventory agent's tool calls succeed
2. Resolve the PermissionDenied error for the customer loyalty agent's discount calculation — likely needs an additional role or a fix in how `discountLogic.py` authenticates
